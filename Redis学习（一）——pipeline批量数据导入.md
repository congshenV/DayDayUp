Redis学习（一）——pipeline批量数据导入
===========================



Redis批量数据导入的方式很多，可以通过python脚本解析文本并使用pipeline批量命令的方式实现，也可以通过批量命令文本+pipeline的形式。

****
	
|Author|葱shen|
|---|---|
|E-mail|yycmessi@163.com



****
## 今日内容预览
* [pipeline简介](#pipeline简介)
* [pipeline批量数据导入](#pipeline批量数据导入)




### pipeline简介
-----------
Redis是一种基于客户端-服务端模型以及请求/响应协议的TCP服务。
这意味着通常情况下一个请求会遵循以下步骤：
>1、客户端向服务端发送一个查询请求，并监听Socket返回，通常是以阻塞模式，等待服务端响应。
>2、服务端处理命令，并将结果返回给客户端。

这一整个执行时间，被称为RTT (Round Trip Time - 往返时间)。当进行大量的命令请求时，就会发现这会给性能带来多大影响。
*例如，如果RTT时间是250毫秒（在一个很慢的连接下），即使服务器每秒能处理100k的请求数，我们每秒最多也只能处理4个请求。*

服务器在请求还未被响应的时候就可以处理新的请求，这样就可以将多个命令发送到服务器，而不用等待回复，最后在一个步骤中读取该答复，这就是管道（pipelining）。
Redis很早就支持管道（pipelining）技术，因此无论你运行的是什么版本，你都可以使用管道（pipelining）操作Redis。


### pipeline批量数据导入
-----------
**重要说明: 使用管道发送命令时，服务器将被迫回复一个队列答复，占用很多内存。所以，如果你需要发送大量的命令，最好是把他们按照合理数量分批次的处理**
简单介绍下第一种批量导入的方式，即python脚本解析文本，并通过pipeline批量导入命令的形式：

  

    #!/usr/bin/env python
    # -*- coding: utf-8 -*-
    
    import os
    import sys
    import pymssql
    import redis
    import time
    
    redis_ip = "127.0.0.1"
    redis_port = 6379
    redis_passwd = "test"
    rpool = redis.ConnectionPool(host = redis_ip, port = redis_port, db = 0, password = redis_passwd)
    redis_cli = redis.Redis(connection_pool=rpool)
    pipeline_redis = redis_cli.pipeline(transaction=False)
    count = 0
    file_name = sys.argv[1]
    f = open(file_name)
    file_content = f.readlines()
    cmd_list = []
    for single_line in file_content:
        field ,uid, val_str = single_line.strip('\n').split('^')
        val = int(val_str)
        key = "KEYHEAD%s" % uid
        count += 1
        pipeline_redis.hset(key, field, val)
        #cmd_list.append([key, field, val])
        if count % 1000 == 0:
            result = pipeline_redis.execute()
            print result
    result = pipeline_redis.execute()
    print result

第二种导入方式，也是比较简单的方法：

    #第一步，将完整的Redis命令写入文本之中，如通过MySQL将数据拼成Redis命令，并生成文本：
    mysql -h127.0.0.1 -P3306 -uroot -proot -e "select concat_ws(' ','hset',key,field,number) from Testdb.testtable;">> tmp.csv
    #第二步，通过redis-cli --pipe导入
    cat tmp.csv | redis-cli -h 127.0.0.1 -p 6379 -a test --pipe
    
**注意：redis-cli中只支持dos格式的换行符 \r\n ，如果你在Linux下、Mac下或者Windows下创建的文件，最好都转个码。没有转码的文件,执行会失败。
下面是转码指令, 只需要在命令后加入要转码的文件即可：**

    unix2dos tmp.csv
    unix2dos: converting file tmp.csv to DOS format...**

