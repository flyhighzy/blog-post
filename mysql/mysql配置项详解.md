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
mysqld.conf:
```
[mysqld]
server-id   = 3158126641
log-slave-updates = 0
binlog-format            = STATEMENT
```

 - core-file

 对 MySQL 来说，由于 core file 中会包含表空间的数据，所以默认情况下为了安全，mysqld 捕获了 SEGV 等信号，崩溃时并不会生成 core file，需要在 my.cnf 或启动参数中加上 core-file。

 - !include

 引用其它conf文件

 - basedir

 mysql安装路径的根目录

- datadir

  mysql数据存放路径

- server-id

  server编号，主要用于数据主从同步时，标识该说一句最初是从哪个server写入的，同时，每一个同步中的slave在master上都对应一个master线程，也是用server-id来标识唯一性的。

- log-slave-updates

  为0时（不启用），slave不把从master同步过来的数据写入自己的binlog。
  只有当需要设置链条式备份时，如 `A -> B -> C`时，B既是slave又是C的master，此时它需要把从A同步过来的数据也写入自己的binlog，才需要启用log-slave-updates。
  启用时，需要同时设置log-bin启用binlog才可以启用。

- binlog-format

  默认值：STATEMENT. 意味着master是把同步库上执行的sql语句同步到slave并在slave执行来达到同步数据的上的。这个选项也支持create, drop等ddl操作的同步。但若sql中依赖了某些系统函数，可能会有从库redo时数据不一致的风险。
  可选：ROW. 即row-based logging，master会把自身表内数据的变化写到binlog中，同步的只是数据，要求表结构一致，且不会造成主从数据不一致的情况。
  可选：MIXED。即默认logging方式为statement，但在某些有数据不一致风险的情况下会自动切换为row-based logging。

  会发生切换的情况：
  1. call UUID()
  2. AUTO_INCREMENT列被更新时，或trigger/stored function被调用时
  3. 创建view时的语句需要row-based备份
  4. UDF被调用时
  5. INSERT DELAYED执行于nontransactional table时。（即当事务失败时需要手动恢复数据的表）
  6. 若statement是按行记日志的，且执行statement的session有创建临时表，则所有临时表都会用row-based来记录，直到这个session中的临时表被删除。
  7. 使用了FOUND_ROWS(),ROW_COUNT()方法
  8. 使用了USER(),CURRENT_USER(),CURRENT_USER方法
  9. 语句中使用了系统变量时，但以下变量只使用在session scope时例外：
    - auto_increment_increment

    - auto_increment_offset

    - character_set_client

    - character_set_connection

    - character_set_database

    - character_set_server

    - collation_connection

    - collation_database

    - collation_server

    - foreign_key_checks

    - identity

    - last_insert_id

    - lc_time_names

    - pseudo_thread_id

    - sql_auto_is_null

    - time_zone

    - timestamp

    - unique_checks
  10. 使用到了mysql db中的log表
  11. 使用了LOAD_FILE()

### tmp dir settings
```
# tmp dir settings
tmpdir                   = /home/xxx/mysql-server
slave-load-tmpdir        = /home/xxx/mysql-server
```

- tmpdir

  存放临时文件目录，若/tmp目录比较小，可以重设这个参数。这个参数可以设置多个路径，冒号分割(unix)或分号分割（windows），mysql会round-robin来使用多个路径

- slave-load-tmpdir

  slave同步数据时创建临时文件的地方，默认与tmpdir相同。

### skip options
```
# skip options
#skip-grant-tables
skip-name-resolve
skip-symbolic-links
skip-external-locking
skip-slave-start
```

- skip-grant-tables

  启动时不加载授权表，这就意味着任务账号用任何密码都能登录mysql，这一般只在忘记管理员密码时可用来修改管理员密码。

- skip-name-resolve

  若禁用，则client在登录时不会进行ip -> host的查找，这要求grant table中所有项均为ip地址，而非host，可提升client登录效率。

- skip-symbolic-links

  禁用符号链接的支持。相反的选项为`symbolic-links`。

  windowns上的表现：
  若启用，则可以创建一个符号链接到真正的数据库目录，命名为`db_name.sym`。

  unix上的表现：
  若启用，则允许使用`INDEX DIRECTORY`或`DATA DIRECTORY`选项把一个MyISAM索引文件或数据文件链接到另一个目录。若该表被删除或重命名，则真实的文件也会被删除或重命名。

- skip-external-locking

  不启用外部锁（系统锁），此选项默认关闭，启用的命令为`external-locking`。使用外部锁很容易造成死锁，特别是在linux这种lockd不能完全工作的系统上。

  若启用，此选项也只对MyISAM表有效。

- skip-slave-start

  告诉slave server在启动时暂不启动slave线程。后续可用`START SLAVE`命令启动slave线程。

### 资源设置
```
# res settings
back_log                 = 50
max_connections          = 1000
max_connect_errors       = 10000
#open_files_limit         = 10240

connect-timeout          = 5
wait-timeout             = 28800
interactive-timeout      = 28800
slave-net-timeout        = 600
net_read_timeout         = 30
net_write_timeout        = 60
net_retry_count          = 10
net_buffer_length        = 16384
max_allowed_packet       = 64M

#
thread_stack             = 192K
thread_cache_size        = 20
thread_concurrency       = 8

# qcache settings
query_cache_type         = 0
query_cache_size         = 32M
#query_cache_size         = 256M
#query_cache_limit        = 2M
#query_cache_min_res_unit = 2K
```












```

#sysdate-is-now



# default settings
# time zone
default-time-zone        = system
character-set-server     = utf8
default-storage-engine   = InnoDB
#default-storage-engine   = MyISAM

# tmp & heap
tmp_table_size           = 512M
max_heap_table_size      = 512M

log-bin                  = mysql-bin
log-bin-index            = mysql-bin.index
max_binlog_size          = 1G
sync-binlog      	 = 1000
binlog_format		 = ROW

relay-log                = relay-log
relay-log-index		 = relay-log.index
sync_relay_log	         = 1000
max_relay_log_size       = 1G

# warning & error log
log-warnings             = 1
log-error                = /home/work/mysql/output/../mysql-server/log/mysql.err

# slow query log
long-query-time          = 1
slow_query_log           = 1
slow_query_log_file      = /home/work/mysql/output/../mysql-server/log/slow.log
#log-queries-not-using-indexes
# general query log
general_log				= 1
general_log_file                      = /home/work/mysql/output/../mysql-server/log/mysql.log

# if use auto-ex, set to 0
relay-log-purge          = 1

# max binlog keeps days
expire_logs_days         = 7

binlog_cache_size        = 1M

# replication
replicate-wild-ignore-table     = mysql.%
replicate-wild-ignore-table     = test.%
# slave_skip_errors=all

replicate-do-db = biz
# sync table
replicate-wild-do-table = biz.task_info

# not sync table
replicate-wild-ignore-table = biz.task_request

key_buffer_size                 = 256M
sort_buffer_size                = 2M
read_buffer_size                = 2M
join_buffer_size                = 8M
read_rnd_buffer_size            = 8M
bulk_insert_buffer_size         = 64M
myisam_sort_buffer_size         = 64M
myisam_max_sort_file_size       = 10G
myisam_repair_threads           = 1
myisam_recover

transaction_isolation           = REPEATABLE-READ

#skip-innodb

innodb_file_per_table
innodb_file_format = Barracuda

#innodb_status_file              = 1
#innodb_open_files              = 2048
innodb_buffer_pool_size         = 2G
innodb_data_home_dir            = /home/work/mysql/output/../mysql-server/var
innodb_data_file_path           = ibdata1:1G:autoextend
innodb_file_io_threads          = 4
innodb_read_io_threads 		= 20
innodb_write_io_threads		= 20
innodb_thread_concurrency       = 16
#innodb_use_native_aio = 1
innodb_flush_log_at_trx_commit  = 1

innodb_log_buffer_size          = 8M
innodb_log_file_size            = 1900M
innodb_log_files_in_group       = 2
innodb_log_group_home_dir       = /home/work/mysql/output/../mysql-server/var

innodb_max_dirty_pages_pct      = 90
innodb_lock_wait_timeout        = 50
#innodb-adaptive-hash-index	= 0
#innodb_compression_failure_threshold_pct = 0
#autocommit			= 0

[mysqldump]
quick
max_allowed_packet              = 64M

[mysql]
disable-auto-rehash
default-character-set           = utf8
connect-timeout                 = 3

[isamchk]
key_buffer = 256M
sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M

[myisamchk]
key_buffer = 256M
sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M

[mysqlhotcopy]
interactive-timeout
```
