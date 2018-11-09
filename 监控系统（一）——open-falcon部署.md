监控系统（一）——open-falcon部署
===========================
本文主要介绍open-falcon的搭建。

****
|Author|葱shen|
|---|---|
|E-mail|yycmessi@163.com

****
## 今日内容预览
* [部署环境](#部署环境)
* [相关软件安装](#相关软件安装)
* [安装open-falcon](#安装open-falcon)
* [安装Dashboard](#安装Dashboard)

### 部署环境
-----------
仅供参考
CentOS：6.9
MySQL：5.7
Redis：3.2.6
git：2.9.5
open-falcon：0.2.1
python：2.6.6
Flask：0.10.1
Jinja2：2.7.2
Werkzeug：0.9.4
gunicorn：19.9.0
pythondateutil：2.2
requests：2.3.0
mysql-python：1.2.5



### 相关软件安装
-----------
1、创建用户及组

    groupadd falcon
    mkdir /data/open-falcon
    useradd -g falcon falcon -b /data/open-falcon -s /bin/bash
    usermod -G falcon falcon
    chown -R falcon /data/open-falcon/
    chgrp -R falcon /data/open-falcon/

2、git安装

    yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel gcc perl-ExtUtils-MakeMaker
    wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.9.5.tar.gz
    su - falcon
    tar -zxvf git-2.9.5.tar.gz
    
    cd /data/open-falcon/git-2.9.5/
    make prefix=/data/open-falcon/Paasapplications/git all
    make prefix=/data/open-falcon/Paasapplications/git install
    ln -s /data/open-falcon/Paasapplications/git/bin/git /usr/bin/git
    echo "export PATH=$PATH:/data/open-falcon/Paasapplications/git/bin" >> /data/open-falcon/.bash_profile

3、go安装

    wget https://studygolang.com/dl/golang/go1.11.linux-amd64.tar.gz
    tar zxvf go1.11.linux-amd64.tar.gz  -C /data/open-falcon/Paasapplications
    vim /data/open-falcon/.bash_profile 添加以下两行
    echo "export GOROOT=/data/open-falcon/Paasapplications/go" >> /data/open-falcon/.bash_profile
    echo "export PATH=$PATH:/data/open-falcon/Paasapplications/go/bin" >> /data/open-falcon/.bash_profile
    echo "export GOPATH=/data/open-falcon/Paasapplications" >> /data/open-falcon/.bash_profile
    
    source /data/open-falcon/.bash_profile
    go version

4、MySQL、Redis安装
略

### 安装open-falcon
-----------
1、编译open-falcon源码，生成二进制部署文件

    mkdir -p $GOPATH/src/github.com/open-falcon
    git clone https://github.com/open-falcon/falcon-plus.git
    切换到falcon用户，记得所有文件夹的权限都是falcon用户的
    su - falcon
    然后make all即可
    
    
    [falcon@127.0.0.1 /data/open-falcon/Paasapplications/src/github.com/open-falcon/falcon-plus]$make all
    go build -o bin/agent/falcon-agent ./modules/agent
    go build -o bin/aggregator/falcon-aggregator ./modules/aggregator
    go build -o bin/graph/falcon-graph ./modules/graph
    go build -o bin/hbs/falcon-hbs ./modules/hbs
    go build -o bin/judge/falcon-judge ./modules/judge
    go build -o bin/nodata/falcon-nodata ./modules/nodata
    go build -o bin/transfer/falcon-transfer ./modules/transfer
    go build -o bin/gateway/falcon-gateway ./modules/gateway
    go build -o bin/api/falcon-api ./modules/api
    go build -o bin/alarm/falcon-alarm ./modules/alarm
    go build -ldflags "-X main.GitCommit=`git rev-parse --short HEAD` -X main.Version=0.2.1" -o open-falcon
        
    单一组件编译
    make agent
    打包
    make package得出二进制tar包
    su - falcon
    mv open-falcon-v0.2.1.tar.gz ~/
    tar -zxvf open-falcon-v0.2.1.tar.gz
    
2、初始化DB

    cd /data/open-falcon/Paasapplications/src/github.com/open-falcon/falcon-plus/scripts/mysql/db_schema/
    mysql -h 127.0.0.1 -u root -p < 1_uic-db-schema.sql
    ...同样导入其他几个sql文件

3、部署各组件及启动

    mv open-falcon-v0.2.1.tar.gz ~/
    tar -zxvf open-falcon-v0.2.1.tar.gz
    修改配置文件，启动
    ./open-falcon start
    
### 安装Dashboard
-----------
1、安装相关依赖

    切换到root用户下
    yum install -y python-virtualenv
    yum install -y python-devel
    yum install -y openldap-devel
    yum install -y mysql-devel
    yum groupinstall "Development tools"   这一步出错了
    --> Finished Dependency Resolution
    Error: Package: subversion-1.9.7-1.x86_64 (jjmatch)
               Requires: libserf-1.so.1()(64bit)
    You could try using --skip-broken to work around the problem
    You could try running: rpm -Va --nofiles --nodigest
    
    
    su - falcon
    cd /data/open-falcon/falcon/
    git clone https://github.com/open-falcon/dashboard.git
    cd dashboard
    virtualenv ./env  ---创建dashboard运行的虚拟环境
    ./env/bin/pip install -r pip_requirements.txt
    (本次执行失败，所以分开执行)
    [falcon@127.0.0.1 ~/dashboard]$cat pip_requirements.txt
    Flask==0.10.1
    Flask-Babel==0.9
    Jinja2==2.7.2
    Werkzeug==0.9.4
    gunicorn==19.9.0
    python-dateutil==2.2
    requests==2.3.0
    mysql-python
    python-ldap
    
    需要逐个安装，需注意版本：
    ./env/bin/pip install Flask==0.10.1
    ...
    
2、修改接口、配置

    vim rrd/config.py
    修改api和数据库配置即可
    
3、启动Dashboard

    前端启动。用于测试
    ./env/bin/python wsgi.py
    通过gunicorn控制
    启动：./control start
    停止：./control stop
    日志：./control tail