LVS学习（一）——初识
===========================
`LVS`为`Linux Virtual Server`（虚拟服务器）的简称。

****
	
|Author|葱shen|
|---|---|
|E-mail|yycmessi@163.com



****
## 今日内容预览
* [LVS是什么](#LVS是什么)
* [LVS能做什么](#LVS能做什么)
* [LVS是怎么做到的](#LVS是怎么做到的)




### LVS是什么
-----------
LVS是一个虚拟的四层路由交换器集群系统，根据目标地址和目标端口实现用户请求转发。



### LVS能做什么
-----------
LVS是一个虚拟的服务器集群系统，可以在UNIX/LINUX平台下实现负载均衡集群功能。



### LVS是怎么做到的
-----------
![LVS实现过程][1]

>用户通过vip向lvs发生请求——>
>>lvs收到连接，PREROUTING链首先会接收到用户请求，判断目标IP确定是本机IP，将数据包发往INPUT链——>
>>>IPVS就是工作在input链上的，IPVS会将用户请求和自己已定义好的集群服务进行比对，如果用户请求的就是定义的集群服务，那么此时IPVS会强行修改数据包里的目标IP地址及端口，并将新的数据包发往POSTROUTING链——>
>>>>POSTROUTING链接收数据包后发现目标IP地址刚好是自己的后端服务器，那么此时通过选路，将数据包最终发送给后端的服务器


  [1]: http://mmbiz.qpic.cn/mmbiz_gif/IP70Vic417DPWibVk21gGnIB9vemsXDep4qDn0n6wCqowFZoDdhaLWdUzAZrPYfKaElaCPAJS0jzc2vKq9cGyjSQ/0?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1