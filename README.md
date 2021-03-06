# go-http-client
go的http请求库参数特别多，有一些默认参数甚至会让你踩坑，我做了几组不同的实验来验证不同使用下的表现

>server1 && server2
>> 服务端，监听一个端口，支持keepalive，并且不做超时限制（这是重点，真有这样的服务器）

>client1
>>全局变量发起请求，不做连接后空闲超时限制，请求连接上会一直存在，请求多个域名，每个域名有一个句柄。

>client2
>>全局变量发起，先不设置空闲时间请求，句柄为ESTAB, 然后设置IdleConnTimeout再请求，一共有2个句柄，句柄都变为TIME-WAIT 
>>全局变量的特点就是不同请求可能要求的超时等设置不同，他们会相互影响。只可在某一固定功能上通用

>client3
>>局部变量，不做连接后空闲超时限制，每个请求一个句柄，如果正好遇到服务器也没有空闲超时限制，那句柄数很快会爆

>client4
>>局部变量，设置SetDeadline。每个请求一个句柄，句柄会正常关闭。不会有问题

>client5
>>局部变量，关闭keepalive功能，请求的句柄使用完马上被销毁。观察返回值每次请求的端口都不同，说明都是重新TCP握手的

>client6
>>全局变量，使用IdleConnTimeout 如果sleep时间小于idle时间则能一直复用句柄，如果sleep时间大于idle时间每次都要新开句柄

>client7
>>全局变量，使用SetDeadline 就算sleep时间小于idle时间在每隔设定时间也会开一新句柄
# 汇总

1. 测试中只要ts.IdleConnTimeout或设置conn.SetDeadline()都可以起到效果, 但IdleConnTimeout更优
2. 根据client-timeout.png中所示，最好要设置3个超时时间（Dialer.Timeout,Client.Timeout,IdleConnTimeout)
3. 根据参考的博文中说明MaxIdleConnsPerHost默认值只有2有时也是不够的，可以设置为1024
4. 有需要时，可以通过DisableKeepAlives:true关闭长连接
5. 即使不同域名也可以使用同一个client变量。当不同业务需要的Transport可能会不同时client变量要分开。

# 参考
* https://blog.csdn.net/bigwhite20xx/article/details/112386441
* https://blog.huati365.com/ab420f833f7ffcfd
* https://wangchujiang.com/linux-command/c/ss.html
* https://www.oschina.net/translate/the-complete-guide-to-golang-net-http-timeouts?print
