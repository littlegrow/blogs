---
title: mac 安装 mysql
date: 2020-5-6 9:00:00
description: "记录mysql安装步骤"
categories:
- mysql
---


### 安装mysql

* 安装命令

```
brew install mysql
```

* 启动

```
mysql.server start
```

* 安全设置，修改密码

```
mysql_secure_installation
```


### 创建数据库，授权用户

* 登录mysql

```
mysql -u root -p
```

输入刚才设置的密码

* 创建数据库

```
create database db_name character set utf8mb4;
```

* 创建用户

```
create user 'username'@'%' identified by 'password';
```

* 用户授权

```
grant all privileges on db_name.* to 'username'@'%';
flush privileges;
```


### 命令行免密码登录

* 创建配置文件

```
cd ~;
touch .my.cnf
```

* 编辑配置文件

```
[client]
host			= 127.0.0.1
user            = *******
password        = *******
port            = 3306
database		= ******
```

* 重启服务

```
brew services restart mysql
```


* 无密码登录mysql

```
➜  ~ mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 20
Server version: 8.0.19 Homebrew

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```










