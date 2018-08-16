### 1：首先通过show full processlist查看正在执行的语句；

或者通过 SELECT * FROM information_schema.processlist ORDER BY TIME DESC 对processlist结果进行筛选。结果如下：
其中time表示执行时间，单位是秒。

 

![img](https://upload-images.jianshu.io/upload_images/6167458-a8f71c2d3242484a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

### 2：如果看到大量重复又耗时的sql，则为慢sql堆积，可以先停服务，或者服务限流，然后kill掉这些慢查询。

接着具体分析这些sql，并且优化sql。必要的话暂停相关业务。

### 3：还有可能是睡眠连接过多，严重消耗mysql服务器资源

wait_timeout, 即可设置睡眠连接超时秒数，如果某个连接超时，会被mysql自然终止。
show global variables like 'wait_timeout' 查看timeout时间。
通过set global wait_timeout=20 改变timeout时间。

具体参考：<https://www.cnblogs.com/kevingrace/p/6226350.html>