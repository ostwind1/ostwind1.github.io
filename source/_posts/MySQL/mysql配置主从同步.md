---
title: mysql配置主从同步
date: 2022-03-22 23:26:00
tags: 
    - 数据库 
    - MySQL
    - 主从同步
category: MySQL
summary: 玩一下主从同步
---

## 0x00 环境准备
* macos 12.1
* docker desktop 3.3.1 ， 科学上网/使用docker国内仓库

## 创建 mysql 容器
### 文件组织结构如下
```
- mysql/
    - conf/
        - my.cnf
- musql_slave/
    - conf/
        - slave.conf
- .env
- docker-compose.yml
```


### 各文件内容如下
> 在这里创建两台机器 一主一丛
#### 1. `docker-compose.yml` 文件
```yml
version: "3"
services: 
  mysql-master:
    image: mysql:8.0
    container_name: db_master
    ports:
      - 3316:3306
    volumes:
      - ${MYSQL_DIR}/data:/var/lib/mysql
      - ${MYSQL_DIR}/conf/my.cnf:/etc/mysql/conf.d/my.cnf
    command:
      - --default_authentication_plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --lower_case_table_names=2
    restart: always 
    env_file: .env
  mysql-slave:
    image: mysql:8.0
    container_name: db_slave
    ports:
      - 3317:3306
    volumes:
      - ${MYSQL_SLAVE_DIR}/data:/var/lib/mysql
      - ${MYSQL_SLAVE_DIR}/conf/slave.cnf:/etc/mysql/conf.d/my.cnf
    command:
      - --default_authentication_plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --lower_case_table_names=2
    restart: always 
    env_file: .env
```

#### 2. `.env`文件
```env
MYSQL_DIR=./mysql
MYSQL_SLAVE_DIR=./mysql_slave
MYSQL_ROOT_PASSWORD=root
MYSQL_USER=test
MYSQL_PASSWORD=test
```
#### 3. 创建主备两台mysql的配置文件
##### 主库的配置文件 `my.cnf`
```cnf
[mysqld]
#主从配置

# server-id 唯一的服务辨识号,数值位于 1 到 2^32-1之间.
# 此值在master和slave上都需要设置.
# 如果 “master-host” 没有被设置,则默认为1, 但是如果忽略此选项,MySQL不会作为master生效.
server-id=1             #服务器 id 

# log-bin 打开二进制日志功能.                                                                               
# 在复制(replication)配置中,作为MASTER主服务器必须打开此项                                                   
# 如果你需要从你最后的备份中做基于时间点的恢复,你也同样需要二进制日志. 
log-bin=mysql-bin       #启用二进制日志并设置文件名称
binlog-do-db=sync-test  #待同步的数据库
binlog-ignore-db=mysql  #不同步的数据

binlog_format = mixed #binlog日志格式，mysql默认采用，如果从服务器slave有别的slave要复制那么该slave也需要这一项 
expire_logs_days = 7 #超过7天的binlog删除 

# 开启gtid
gtid_mode=on
enforce-gtid-consistency=true
```

##### 备库的 配置文件 `slave.cnf`
```cnf
[mysqld]
#主从配置
server-id=4                 #服务器 id 
log-bin=mysql-bin           #二进制文件存放路径
binlog_format = mixed #binlog日志格式，mysql默认采用，如果从服务器slave有别的slave要复制那么该slave也需要这一项 

# replicate-do-db 需要做复制的数据库,如果复制多个数据库，重复设置这选项即可master上不需要此项，slave上需要  
replicate-do-db=sync-test        #待同步的数据库

replicate-ignore-db=mysql   #不同步的数据
relay_log=edu-mysql-relay-bin  ## relay_log配置中继日志

# 开启gtid
gtid_mode=on
enforce-gtid-consistency=true

# 从库设置只读属性
read_only=1
```

## 创建容器
在 docker-compose.yml 文件同级下使用 `docker-compose up -d` 命令启动容器

## 配置主从同步
容器创建成功后， 连接到备库上， 执行以下SQL
```SQL
/* 因为开启了gtid 因此我们这里无需指明 master_log_file 和 master_log_pos */
change master to master_host='db_master' , master_port=3306,  master_user='root', master_password='root',  master_auto_position = 1;

start slave;
```

执行成功后可通过查看从库状态， 其中 `Slave_IO_Running` 和 `Slave_SQL_Running` 均为 Yes 表示主从建立成功。
```SQL
show slave status;
```

此时在主mysql的`sync-test`库上执行的变更会自动同步到备库