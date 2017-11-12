# pybackup使用文档
pybackup源自于对线上备份脚本的改进和对备份情况的监控需求.

原本生产库的备份是通过shell脚本调用mydumper,之后再将备份通过rsync传输到备份机.

想要获取备份状态,时间,rsync传输时间等信息只能通过解析日志.

pybackup由python编写,调用mydumper和rsync,将备份信息存入数据库中,后期可以通过grafana图形化展示和监控备份

## 参数说明
```
pybackup.py mydumper ARG_WITH_NO_--... (([--no-rsync] [--no-history]) | [--only-backup])
```
除了最后三个参数,使用的所有参数和mydumper -h中列出的参数相同. 只不过目前只支持长选项,并且不带'--'
列:
```
./pybackup.py mydumper password=fanboshi database=fandb outputdir=/data4/recover/pybackup/2017-11-12 logfile=/data4/recover/pybackup/bak.log verbose=3
```
可以使用`./pybackup.py mydumper help`查看mydumper帮助信息

--no-rsync
不使用rsync传输

--no-history
不记录备份信息到数据库

--only-backup
等价于同时使用--no-rsync和--no-history . 不能与--no-rsync或--no-history同时使用

## 配置文件说明
配置文件为pbackup.conf
```
[root@localhost pybackup]# less pybackup.conf 
[CMDB]            --存储备份信息的数据库配置
db_host=localhost
db_port=3306
db_user=root
db_passwd=fanboshi
db_use=cmdb

[TDB]              --需要备份的数据库配置
db_host=localhost
db_port=3306
db_user=root
db_passwd=fanboshi
db_use=information_schema
db_list=test,fandb,union_log_ad_% --指定需要备份的数据库,可以使用mysql支持的通配符. 如果想要备份所有数据库则填写%

[rsync]
password_file=/data4/recover/pybackup/rsync.sec    --等同于--password-file
dest=platform@182.92.83.238/db_backup/106.3.130.84 --传输到哪个目录
address=                                           --使用的网卡.可以为空不指定
```
注意
```
[TDB]
db_list=fan,bo,shi           代表备份fan,bo,shi三个数据库
db_list=!fan,!bo,!shi        代表不备份fan,bo,shi三个数据库
db_list=%                    代表备份所有数据库
db_list=!fan,bo              不支持
```

备份信息示例
```
*************************** 3. row ***************************
               id: 3
            bk_id: 7d4085d8-c76f-11e7-ace3-005056b15d9c
        bk_server: 106.3.130.84
       start_time: 2017-11-12 14:05:31
         end_time: 2017-11-12 14:06:26
     elapsed_time: 54
      is_complete: N,Y,Y,Y,Y,Y,Y,Y,Y,Y
          bk_size: 2.2G
           bk_dir: /data4/recover/pybackup/2017-11-12
   transfer_start: NULL
     transfer_end: NULL
 transfer_elapsed: NULL
transfer_complete: N/A (local backup)
      remote_dest: platform@182.92.83.238/db_backup/106.3.130.84/
    master_status: mysql-bin.000036,25255,
     slave_status: Not a slave
     tool_version: mydumper 0.9.2, built against MySQL 5.5.53
   server_version: 5.7.18-log
       bk_command: mydumper --password=supersecrect --outputdir=/data4/recover/pybackup/2017-11-12 --verbose=3 database=information_schema,PERCONA_SCHEMA,cmdb,fandb,mysql,performance_schema,sys,test,union_log_ad_201710_db,union_log_ad_201711_db
```
建库建表语句
```
create database cmdb;
create table user_backup(
id int auto_increment not null primary key,
bk_id char(36) not null unique key,
bk_server varchar(15) not null,
start_time datetime not null,
end_time datetime not null,
elapsed_time int not null,
is_complete varchar(30) not null,
bk_size varchar(10) not null,
bk_dir varchar(200) not null,
transfer_start datetime,
transfer_end datetime,
transfer_elapsed int,
transfer_complete varchar(20) not null,
remote_dest varchar(200) not null,
master_status varchar(200) not null,
slave_status varchar(200) not null,
tool_version varchar(200) not null,
server_version varchar(200) not null,
bk_command varchar(400) not null
) engine=innodb character set utf8 collate utf8_general_ci;
```
