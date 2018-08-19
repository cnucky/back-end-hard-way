## 1.Reactor模式介绍

Reactor模式是事件驱动模型，有一个或多个并发输入源，有一个Service Handler，有多个Request Handlers；这个Service Handler会同步的将输入的请求（Event）多路复用的分发给相应的Request Handler。从结构上，这有点类似生产者消费者模式，即有一个或多个生产者将事件放入一个Queue中，而一个或多个消费者主动的从这个Queue中Poll事件来处理；而Reactor模式则并没有Queue来做缓冲，每当一个Event输入到Service Handler之后，该Service Handler会主动的根据不同的Event类型将其分发给对应的Request Handler来处理。

![reactor类图](..\resource\reactor类图.jpg)

该图为Reactor模型，实现Reactor模式需要实现以下几个类：

1）EventHandler：事件处理器，可以根据事件的不同状态创建处理不同状态的处理器；

2）Handle：可以理解为事件，在网络编程中就是一个Socket，在数据库操作中就是一个DBConnection；

3）InitiationDispatcher：用于管理EventHandler，分发event的容器，也是一个事件处理调度器，Tomcat的Dispatcher就是一个很好的实现，用于接收到网络请求后进行第一步的任务分发，分发给相应的处理器去异步处理，来保证吞吐量；

4）Demultiplexer：阻塞等待一系列的Handle中的事件到来，如果阻塞等待返回，即表示在返回的Handle中可以不阻塞的执行返回的事件类型。这个模块一般使用操作系统的select来实现。在Java NIO中用Selector来封装，当Selector.select()返回时，可以调用Selector的selectedKeys()方法获取Set<SelectionKey>，一个SelectionKey表达一个有事件发生的Channel以及该Channel上的事件类型。

![reactor时序图](..\resource\reactor时序图.jpg)

1）初始化InitiationDispatcher，并初始化一个Handle到EventHandler的Map。

2）注册EventHandler到InitiationDispatcher中，每个EventHandler包含对相应Handle的引用，从而建立Handle到EventHandler的映射（Map）。

3）调用InitiationDispatcher的handle_events()方法以启动Event Loop。在Event Loop中，调用select()方法（Synchronous Event Demultiplexer）阻塞等待Event发生。

4）当某个或某些Handle的Event发生后，select()方法返回，InitiationDispatcher根据返回的Handle找到注册的EventHandler，并回调该EventHandler的handle_events()方法。

5）在EventHandler的handle_events()方法中还可以向InitiationDispatcher中注册新的Eventhandler，比如对AcceptorEventHandler来，当有新的client连接时，它会产生新的EventHandler以处理新的连接，并注册到InitiationDispatcher中。

## 2.Reactor Java实现

介绍了这些估计大部分同学还是云里雾里，下面我们直接通过代码来实现一个简单的Reactor模式，Reactor模式相对于其它模式来说，就算是最简单的实现也涉及到不少类，这个例子只是为了体现该模式，所以没有实质的业务功能，如果要实现具体业务功能可以自行添加相关代码，网上很多例子都是基于Java NIO的一些类来实现的Reactor模式，这里我们实现没有依赖任何的NIO类库，这主要是方便大家理解该模式，性能上肯定是不如Netty这些NIO框架封装的要好，这样可以方便大家去学习这种模式的思想，该模式也不一定非用在NIO的地方，也可以用在其他的非阻塞需求的业务场景。

实现Reactor我们需要实现以下几个类：

**InputSource:** 外部输入类，用来表示需要reactor去处理的原始对象

**Event**: reactor模式的事件类，可以理解为将输入原始对象根据不同状态包装成一个事件类，reactor模式里处理的斗士event事件对象

**EventType**: 枚举类型表示事件的不同类型

**EventHandler**: 处理事件的抽象类，里面包含了不同事件处理器的公共逻辑和公共对象

**AcceptEventHandler\ReadEventhandler等**: 继承自EventHandler的具体事件处理器的实现类，一般根据事件不同的状态来定义不同的处理器

**Dispatcher**: 事件分发器，整个reactor模式解决的主要问题就是在接收到任务后根据分发器快速进行分发给相应的事件处理器，不需要从开始状态就阻塞

**Selector**: 事件轮循选择器，selector主要实现了轮循队列中的事件状态，取出当前能够处理的状态

**Acceptor**:reactor的事件接收类，负责初始化selector和接收缓冲队列

**Server**:负责启动reactor服务并启动相关服务接收请求

这个例子没有实现什么具体业务逻辑，只是把reactor模式的框架给实现了，大家如果要实现基于reactor模式的业务功能，可以自己定义InputSource、定义事件的不同状态、定义事件处理器，目前大家用的比较多的各种NIO框架就是reactor模式的很好实践，其实reactor模式不一定用在NIO上，也可以用在其他场合，这就要各位发挥自己的想象了，reactor只是一种设计模式，是一种基于事件响应的模式，大家心里一定要理解这点，好了下面开始贴代码。

**InputSource.java**

```
/**
 * @Author: feiweiwei
 * @Description: 输入对象，reactor模式中处理的原始输入对象
 * @Created Date: 10:50 17/10/12.
 * @Modify by:
 */
public class InputSource {
    private Object data;
    private long id;

    public InputSource(Object data, long id) {
        this.data = data;
        this.id = id;
    }

    @Override
    public String toString() {
        return "InputSource{" +
                "data=" + data +
                ", id=" + id +
                '}';
    }
}
```

**Event.java**

```
/**
 * @Author: feiweiwei
 * @Description: reactor模式中内部处理的event类
 * @Created Date: 11:03 17/10/12.
 * @Modify by:
 */
public class Event {
    private InputSource source;
    private EventType type;

    public InputSource getSource() {
        return source;
    }

    public void setSource(InputSource source) {
        this.source = source;
    }

    public EventType getType() {
        return type;
    }

    public void setType(EventType type) {
        this.type = type;
    }
}
```

**EventType**

```
/**
 * @Author: feiweiwei
 * @Description:
 * @Created Date: 11:05 17/10/12.
 * @Modify by:
 */
public enum EventType {
    ACCEPT,
    READ,
    WRITE;
}
```

**EventHandler.java**

```
/**
 * @Author: feiweiwei
 * @Description: event处理器的抽象类
 * @Created Date: 11:26 17/10/12.
 * @Modify by:
 */
public abstract class EventHandler {

    private InputSource source;
    public abstract void handle(Event event);

    public InputSource getSource() {
        return source;
    }

    public void setSource(InputSource source) {
        this.source = source;
    }
}
```

**AcceptEventHandler.java**

```
/**
 * @Author: feiweiwei
 * @Description: ACCEPT事件处理器
 * @Created Date: 11:28 17/10/12.
 * @Modify by:
 */
public class AcceptEventHandler extends EventHandler {
    private Selector selector;

    public AcceptEventHandler(Selector selector) {
        this.selector = selector;
    }

    @Override
    public void handle(Event event) {
        //处理Accept的event事件
        if (event.getType() == EventType.ACCEPT) {

            //TODO 处理ACCEPT状态的事件

            //将事件状态改为下一个READ状态，并放入selector的缓冲队列中
            Event readEvent = new Event();
            readEvent.setSource(event.getSource());
            readEvent.setType(EventType.READ);

            selector.addEvent(readEvent);
        }
    }
}
```

**Dispatcher.java**

```
/**
 * @Author: feiweiwei
 * @Description: reactor模式中Dispatcher类，负责event的分发和eventHandler的维护
 * @Created Date: 12:09 17/10/12.
 * @Modify by:
 */
public class Dispatcher {
    //通过ConcurrentHashMap来维护不同事件处理器
    Map<EventType, EventHandler> eventHandlerMap = new ConcurrentHashMap<EventType, EventHandler>();
    //本例只维护一个selector负责事件选择，netty为了保证性能实现了多个selector来保证循环处理性能，不同事件加入不同的selector的事件缓冲队列
    Selector selector;

    Dispatcher(Selector selector) {
        this.selector = selector;
    }

    //在Dispatcher中注册eventHandler
    public void registEventHandler(EventType eventType, EventHandler eventHandler) {
        eventHandlerMap.put(eventType, eventHandler);

    }

    public void removeEventHandler(EventType eventType) {
        eventHandlerMap.remove(eventType);
    }

    public void handleEvents() {
        dispatch();
    }

    //此例只是实现了简单的事件分发给相应的处理器处理，例子中的处理器都是同步，在reactor模式的典型实现NIO中都是在handle异步处理，来保证非阻塞
    private void dispatch() {
        while (true) {
            List<Event> events = selector.select();

            for (Event event : events) {
                EventHandler eventHandler = eventHandlerMap.get(event.getType());
                eventHandler.handle(event);
            }
        }
    }
}
```

**Selector.java**

```
package com.monkey01.reactor;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

/**
 * @Author: feiweiwei
 * @Description: reactor模式中的Demultiplexer类，提供select（）方法用于从缓冲队列中查找出符合条件的event列表
 * @Created Date: 11:09 17/10/12.
 * @Modify by:
 */
public class Selector {
    //定义一个链表阻塞queue实现缓冲队列，用于保证线程安全
    private BlockingQueue<Event> eventQueue = new LinkedBlockingQueue<Event>();
    //定义一个object用于synchronize方法块上锁
    private Object lock = new Object();

    List<Event> select() {
        return select(0);
    }

    //
    List<Event> select(long timeout) {
        if (timeout > 0) {
            if (eventQueue.isEmpty()) {
                synchronized (lock) {
                    if (eventQueue.isEmpty()) {
                        try {
                            lock.wait(timeout);
                        } catch (InterruptedException e) {
                        }
                    }
                }

            }
        }
        //TODO 例子中只是简单的将event列表全部返回，可以在此处增加业务逻辑，选出符合条件的event进行返回
        List<Event> events = new ArrayList<Event>();
        eventQueue.drainTo(events);
        return events;
    }

    public void addEvent(Event e) {
        //将event事件加入队列
        boolean success = eventQueue.offer(e);
        if (success) {
            synchronized (lock) {
                //如果有新增事件则对lock对象解锁
                lock.notify();
            }

        }
    }

}
```

**Acceptor.java**

```
package com.monkey01.reactor;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

/**
 * @Author: feiweiwei
 * @Description:
 * @Created Date: 13:36 17/10/12.
 * @Modify by:
 */
public class Acceptor implements Runnable{
    private int port; // server socket port
    private Selector selector;

    // 代表 serversocket，通过LinkedBlockingQueue来模拟外部输入请求队列
    private BlockingQueue<InputSource> sourceQueue = new LinkedBlockingQueue<InputSource>();

    Acceptor(Selector selector, int port) {
        this.selector = selector;
        this.port = port;
    }

    //外部有输入请求后，需要加入到请求队列中
    public void addNewConnection(InputSource source) {
        sourceQueue.offer(source);
    }

    public int getPort() {
        return this.port;
    }

    public void run() {
        while (true) {

            InputSource source = null;
            try {
                // 相当于 serversocket.accept()，接收输入请求，该例从请求队列中获取输入请求
                source = sourceQueue.take();
            } catch (InterruptedException e) {
                // ignore it;
            }

            //接收到InputSource后将接收到event设置type为ACCEPT，并将source赋值给event
            if (source != null) {
                Event acceptEvent = new Event();
                acceptEvent.setSource(source);
                acceptEvent.setType(EventType.ACCEPT);

                selector.addEvent(acceptEvent);
            }

        }
    }
}
```

**Server**

```
public class Server {
    Selector selector = new Selector();
    Dispatcher eventLooper = new Dispatcher(selector);
    Acceptor acceptor;

    Server(int port) {
        acceptor = new Acceptor(selector, port);
    }

    public void start() {
        eventLooper.registEventHandler(EventType.ACCEPT, new AcceptEventHandler(selector));
        new Thread(acceptor, "Acceptor-" + acceptor.getPort()).start();
        eventLooper.handleEvents();
    }
}
```

## 3.Reactor优缺点

Reactor模式的核心是解决多请求问题，如果有特别多的请求同时发生，不会因为线程池被短时间占满而拒绝服务。我们一般实现多请求的模块，会采用线程池的实现方案，这种方案对于并发量不是特别大的场景是足够用的，一般来说单机tps1000以下都是够用的，但是线程池方案的最大缺点就是，如果瞬间有大并发则会一下子耗满线程，整个服务陷入阻塞中，后续请求将无法接入。基于Reactor模式实现的方案，会有一个Dispatcher先接收event，然后快速分发给相应的耗时eventHandler处理器去处理，这样就不会阻塞请求的接收。

Reactor模式和生产者消费者模式最大的区别在于，生产者消费者模式是基于队列的实现，能够解决生产端和消费端处理速度不同步的问题，queue可以基于Java Queue或者基于现有的MQ产品来实现；而Reactor模式是基于事件驱动模型，当接收到请求后会将请求封装成事件，并将事件分发给相应处理事件的Handler，handler处理完成后将事件状态修改为下一个状态，再由Reactor将事件分发给能够处理下一个状态的handler进行处理。两个模式解决的问题都是类似的，只是实现起来有所区别，这是我个人的理解。

介绍了那么多，Reactor模式的优点很明显，解耦、提升复用性、模块化、可移植性、事件驱动、细力度的并发控制等。Reactor模式的缺点也很明显，模型复杂，因为涉及到内部回调，多线程处理，不容易调试；需要操作系统底层支持，这就导致不同操作系统可能会产生不一样的结果。所以总的来说如果并发要求不是那么高，使用传统的阻塞线程池模型足够了，而且调试、查问题都会简单很多；如果我们的使用场景是会产生瞬时大并发，可以使用Reactor模式来实现，目前大部分的NIO框架或者容器都是实现了Reactor模式，Tomcat、Jetty的NIO都是实现了Reactor模式，Netty和Mina是两套NIO的框架，也分别对Java NIO进行了二次封装实现了Reactor模式。

 

 

 转载自：https://www.jianshu.com/p/188ef8462100

 

 