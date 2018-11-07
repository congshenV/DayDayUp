Redis学习（二）——python多线程方式快速遍历集群
===========================



Redis单实例的遍历我们都知道，使用scan就可以了。但是对于集群，遍历功能的支持就不是这么友好了，所以本次我就想到用单实例scan+多线程的形式去实现集群的快速遍历。

****
	
|Author|葱shen|
|---|---|
|E-mail|yycmessi@163.com



****
## 今日内容预览
* [scan功能简介](#scan功能简介)
* [python多线程简介](#python多线程简)
* [scan+多线程集群遍历实现](#scan+多线程集群遍历实现)




### scan功能简介
-----------
redis全库遍历key使用的是keys 命令，而keys命令会造成阻塞，所以出现了scan增量式迭代的命令来支持非阻塞遍历。
对于SCAN这类增量式迭代命令来说，有可能在增量迭代过程中，集合元素被修改，对返回值无法提供完全准确的保证。
SCAN命令是一个基于游标的迭代器。这意味着命令每次被调用都需要使用上一次这个调用返回的游标作为该次调用的游标参数，以此来延续之前的迭代过程

当SCAN命令的游标参数被设置为 0 时， 服务器将开始一次新的迭代， 而当服务器向用户返回值为 0 的游标时， 表示迭代已结束。
SCAN命令的返回值 是一个包含两个元素的数组， 第一个数组元素是用于进行下一次迭代的新游标， 而第二个数组元素则是一个数组， 这个数组中包含了所有被迭代的元素。

在第二次调用 SCAN 命令时， 命令返回了游标 0 ， 这表示迭代已经结束， 整个数据集已经被完整遍历过了。
full iteration ：以 0 作为游标开始一次新的迭代， 一直调用 SCAN 命令， 直到命令返回游标 0 ， 我们称这个过程为一次完整遍历。

scan的两个绝对：
>1、从完整遍历开始直到完整遍历结束期间， 一直存在于数据集内的所有元素都会被完整遍历返回； 这意味着， 如果有一个元素， 它从遍历开始直到遍历结束期间都存在于被遍历的数据集当中， 那么 SCAN 命令总会在某次迭代中将这个元素返回给用户。
2、同样，如果一个元素在开始遍历之前被移出集合，并且在遍历开始直到遍历结束期间都没有再加入，那么在遍历返回的元素集中就不会出现该元素。

两个不足
>1、同一个元素可能会被返回多次。 处理重复元素的工作交由应用程序负责， 比如说， 可以考虑将迭代返回的元素仅仅用于可以安全地重复执行多次的操作上。
2、如果一个元素是在迭代过程中被添加到数据集的， 又或者是在迭代过程中从数据集中被删除的， 那么这个元素可能会被返回， 也可能不会。

类似于KEYS 命令，增量式迭代命令通过给定 MATCH 参数的方式实现了通过提供一个 glob 风格的模式参数， 让命令只返回和给定模式相匹配的元素。

MATCH功能对元素的模式匹配工作是在命令从数据集中取出元素后和向客户端返回元素前的这段时间内进行的， 所以如果被迭代的数据集中只有少量元素和模式相匹配， 那么迭代命令或许会在多次执行中都不返回任何元素。

### python多线程简介
-----------
setDaemon(True)将线程声明为守护线程，默认情况下（其实就是setDaemon(False)），主线程执行完自己的任务以后，就退出了，此时子线程会继续执行自己的任务，直到自己的任务结束。当我们使用setDaemon(True)方法，设置子线程为守护线程时，主线程一旦执行结束，则全部线程全部被终止执行，可能出现的情况就是，子线程的任务还没有完全执行结束，就被迫停止。
此时join的作用就凸显出来了，join所完成的工作就是线程同步，即主线程任务结束之后，进入阻塞状态，一直等待其他的子线程执行结束之后，主线程在终止
start()开始线程活动。
join（）的作用是，在子线程完成运行之前，这个子线程的父线程将一直被阻塞。
注意:  join()方法的位置是在for循环外的，也就是说必须等待for循环里的两个进程都结束后，才去执行主进程。
join有一个timeout参数：

当设置守护线程时，含义是主线程对于子线程等待timeout的时间将会杀死该子线程，最后退出程序。所以说，如果有10个子线程，全部的等待时间就是每个timeout的累加和。简单的来说，就是给每个子线程一个timeout的时间，让他去执行，时间一到，不管任务有没有完成，直接杀死。
没有设置守护线程时，主线程将会等待timeout的累加和这样的一段时间，时间一到，主线程结束，但是并没有杀死子线程，子线程依然可以继续执行，直到子线程全部结束，程序退出。
    class threading.Thread()说明：

     
    
    class threading.Thread(group=None, target=None, name=None, args=(), kwargs={})
    
    This constructor should always be called with keyword arguments. Arguments are:
    
    　　group should be None; reserved for future extension when a ThreadGroup class is implemented.
    
    　　target is the callable object to be invoked by the run() method. Defaults to None, meaning nothing is called.
    
    　　name is the thread name. By default, a unique name is constructed of the form “Thread-N” where N is a small decimal number.
    
    　　args is the argument tuple for the target invocation. Defaults to ().
    
    　　kwargs is a dictionary of keyword arguments for the target invocation. Defaults to {}.
    
    If the subclass overrides the constructor, it must make sure to invoke the base class constructor (Thread.__init__()) before doing 
    
    anything else to the thread.

### scan+多线程集群遍历实现
-----------
实现集群遍历，field_1值为多少，那么field_2的值便增多少功能，同时记录field_2的增量值及结果值。
 

    #encoding: utf-8
    import redis
    import sys
    import datetime
    import time
    import threading
    import os
    all_instances = ["127.0.0.1_6379","127.0.0.1_6370","127.0.0.1_6371"]
    
    def main_process(rip, rport):
        print 'thread %s is running...\n' % threading.current_thread().name
        rpool = redis.ConnectionPool(host = rip, port = rport, db = 0, password = 'test')
        redis_cli = redis.Redis(connection_pool=rpool)
        redis_pip = redis_cli.pipeline(transaction=False)
        role = redis_cli.info()["role"]
        if role == "slave":
            return
        log_file_name = "test_%s.csv.%s" % (threading.current_thread().name, time.strftime("%Y%m%d_%H", time.localtime()))
        log_file_name = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), "Logs", log_file_name)
        key_list = []
        pipe_len = 0
        for key in redis_cli.scan_iter(match='*',count=1000):
            redis_pip.hget(key, 'field_1')
            key_list.append(key)
            pipe_len += 1
            if pipe_len % 1000 == 0:
                log_file_fd = open(log_file_name, 'a')
                pip_result = redis_pip.execute()
                key_val_list = []
                for index in range(len(pip_result)):
                    if pip_result[index] and int(pip_result[index]) > 0:
                        redis_pip.hincrby(key_list[index], 'field_2', int(pip_result[index]))
                        key_val_list.append([key_list[index], int(pip_result[index])])
                pip_result = redis_pip.execute()
                for index in range(len(pip_result)):
                    log_file_fd.write("%s,%d,%s\n" % (key_val_list[index][0], key_val_list[index][1], pip_result[index]))
                key_list = []
                log_file_fd.close()
            
        log_file_fd = open(log_file_name, 'a')
        pip_result = redis_pip.execute()
        key_val_list = []
        for index in range(len(pip_result)):
            if pip_result[index] and int(pip_result[index]) > 0:
                redis_pip.hincrby(key_list[index], 'field_2', int(pip_result[index]))
                key_val_list.append([key_list[index], int(pip_result[index])])
        pip_result = redis_pip.execute()
        for index in range(len(pip_result)):
            log_file_fd.write("%s,%d,%s\n" % (key_val_list[index][0], key_val_list[index][1], pip_result[index]))
        log_file_fd.close()

    if __name__ == "__main__":
        start_time = time.time()
        print 'thread %s is running...\n' % threading.current_thread().name
        threads = []
        for instance in all_instances:
            instance_ip, instance_port = instance.split('_')
            cur_thread = threading.Thread(target=main_process, args=(instance_ip, instance_port), name=instance)
            threads.append(cur_thread)
        for eve_t in threads:
            eve_t.setDaemon(True)
            eve_t.start()
        for eve_t in threads:
            eve_t.join()
        print '主线程-%s结束了！用时:%d\n' % (threading.current_thread().name, time.time()-start_time)