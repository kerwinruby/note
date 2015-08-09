#基于CentOS6.6 的 PostgreSQL master-slave部署文档 
###1. 前言
#### 1.1. 目的
 - 保持master上的数据库自动同步到各slave。
 - 当一台电脑坏掉时，数据不会丢失。
 - salve利用本机资源、避免master主机的负载。
 - salve只能只读操作、避免更改数据库。
#### 1.2. 准备环境
 - linux : centOS6.6
 - master ip：192.168.33.10
 - salve ip：194.168.33.11（slave可设多个）
 - postgreSQL: 9.4.4
###2. 步骤
####2.1. 安装postgreSQL(master、slave)
    sudo yum install -y http://yum.postgresql.org/9.4/redhat/rhel-6-x86_64/pgdg-centos94-9.4-1.noarch.rpm
    sudo yum install -y postgresql94-server postgresql94-contrib postgresql94

####2.2. 修改postgreSQL的数据目录(master、slave)
    sudo vi /etc/init.d/postgresql-9.4

将

    PGDATA=/var/lib/pgsql/9.4/data
    PGLOG=/var/lib/pgsql/9.4/pgstartup.log
    PGUPLOG=/var/lib/pgsql/$PGMAJORVERSION/pgupgrade.log

改

    PGDATA=/data/pgsql/9.4/data
    PGLOG=/data/pgsql/9.4/pgstartup.log
    PGUPLOG=/data/pgsql/$PGMAJORVERSION/pgupgrade.log

####2.3. 初始化数据库(master、slave)

    sudo service postgresql-9.4 initdb
    sudo chkconfig postgresql-9.4 on
    sudo service postgresql-9.4 start

####2.4. 在master服务器上创建用户replica，命令如下：
 
    sudo su postgres
    createuser --replication -s -W -l replica
    sudo vi /data/pgsql/9.4/data/postgresql.conf

 将对应项改为如下

    wal_level = hot_standby  # 允许用户replica只读查询
    wal_keep_segments = 256  # 日志文件个数，每个大小是16M
    max_wal_senders = 3  # slave最多个数
####2.5. 添加可访问的网段(master)

    sudo vi /data/pgsql/9.4/data/pg_hba.conf
    host    all     all     192.168.33.0/24 md5 #与主从配置无关，允许局域网内电脑连接数据库
    host    replication     replica 194.168.33.0/24 md5     #允许slave通过用户replica连接
####2.6. 重启master的postgreSQL


