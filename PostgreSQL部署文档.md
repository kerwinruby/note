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
 - salve ip：192.168.33.11（slave可设多个）
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
    psql -c "CREATE USER replica replication superuser login password 'replica'"
    exit
    sudo vi /data/pgsql/9.4/data/postgresql.conf

 将对应项改为如下

    wal_level = hot_standby  # 允许用户replica只读查询
    wal_keep_segments = 256  # 日志文件个数，每个大小是16M
    max_wal_senders = 3  # slave最多个数
    
####2.5. 添加可访问的网段(master)

    sudo vi /data/pgsql/9.4/data/pg_hba.conf
    host    all     all     192.168.33.0/24 md5 #与主从配置无关，允许局域网内电脑连接数据库
    host    replication     replica 192.168.33.11/24 md5     #允许slave通过用户replica连接

####2.6. 重启master的postgreSQL

    sudo service postgresql-9.4 restart	

####2.7. slave配置

    sudo su postgres
    cd ~
    pg_basebackup -D /data/pgsql/9.4/data -Fp -Xs -v -P -h 192.168.33.10 -p 5432 -U replica
    cd 9.4/data
    echo > recovery.conf <<EOF
        standby_mode = 'on'
        primary_conninfo = 'host=192.168.33.10 port=5432 user=replica password=replica'
        trigger_file = '/data/pgsql/9.4/data/failover.trigger.5432.trigger.5432'
    EOF
    exit
    sudo service postgresql-9.4 start
###3. 验证
####3.1 在master上执行

    psql
    create table rep_test(test varchar(40));
    insert into rep_test values ('data one');
    insert into rep_test values ('hello');

####3.2. 在slave上执行

    psql
    select * rep_test
如果能查出master上插入的两条记录，恭喜您，master-slave配置成功。

####3.3. 验证slave的只读属性

    insert into rep_test values ('slave_insert_fail');
    FEHLER:  INSERT kann nicht in einer Read-Only-Transaktion ausgeführt werden

####3.4. 查看同步状态(master)

    select * from pg_stat_replication;

