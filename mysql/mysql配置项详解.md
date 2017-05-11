最近在负责搭建mysql主从同步集群，顺便把my.cnf中涉及到的配置项依次解析一下含义。

## client
```
[client]
port                     = 3306
socket                   = /tmp/mysql.sock
```
配置client连接选项，端口是mysql server端口，socket为连接时使用的socket.

## mysqld

### 基本配置
```
[mysqld]
core-file
!include ../mysql-server/etc/mysqld.cnf
port                     = 3306
socket                   = /home/xxxx/mysql-server/tmp/mysql.sock
pid-file                 = /home/xxxx/mysql-server/var/mysql.pid
basedir                  = /home/xxxx/mysql-server
datadir                  = /home/xxxx/mysql-server/var
```

 - core-file

 对 MySQL 来说，由于 core file 中会包含表空间的数据，所以默认情况下为了安全，mysqld 捕获了 SEGV 等信号，崩溃时并不会生成 core file，需要在 my.cnf 或启动参数中加上 core-file。

 - !include

 引用其它conf文件

 - basedir

 mysql安装路径的根目录

  - datadir

  mysql数据存放路径

  未完待续
