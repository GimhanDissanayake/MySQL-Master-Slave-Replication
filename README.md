# MySQL Master Slave Replication 
![](https://github.com/GimhanDissanayake/MySQL-Master-Slave-Replication/blob/main/mysql%20drw1.PNG)

MySQL replication is a process that allows you to easily maintain multiple copies of a MySQL data by having them copied automatically from a master to a slave database(db). Here I am using one master db replicate to a single slave db.

## Advantages:

1. Facilitate backup from slave database 
2. A way to analyze or generate reports without using the main database
3. Scale out the database

## Pre-requisites: 

- Two Linux Servers with sudo privileges (one of the master server and and one of the slave)

### private IP of master and slave server
```
master ip - 172.16.0.11
slave ip - 172.16.0.22
```
# Configure Master Server

### Install mysql-server and mysql-client
```
apt update

apt install mysql-server mysql-client -y 

mysql_secure_installation

```
### Configure root user
```
mysql
------
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'admin';

FLUSH PRIVILEGES;

SELECT user,authentication_string,plugin,host FROM mysql.user;

exit;
```
### To Connet to mysql without entering passwd each time
```
vim .my.cnf
---------------
[client]
user=root
password=admin
```
### Create a database to copy
```
mysql
-------
show schemas;

create database db1;

use db1;

create table tb1 (name varchar(10));

exit;
```

### Edit Configuration file - master server
```
vim  /etc/mysql/mysql.conf.d/mysqld.cnf
---------------------------------------------------
bind-address=0.0.0.0

[mysqld]
server-is=1
log-bin=mysql-bin
```

### Restart and Check Status
```
systemctl restart mysql
systemctl status mysql
```

### Check Lisetening Port
```
apt install net-tools -y

netstat -tlpn | grep mysql
```

### Add Slave user with permissions 
```
mysql
--------
grant replication slave on *.* to 'slave'@'172.16.0.22' identified by 'slave_passwd';

flush privileges;

show grants for 'slave'@'172.16.0.22';
```

### Take Current Backup of database  
```
mysql
--------
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
----------------------------------------------------------------------
# notedown [ master_log_file , master_log_pos ]
----------------------------------------------------------------------
# open duplicate cli window for master and take backup
----------------------------------------------------------------------
mysqldump -u root -p --opt db1 > db1.sql

sftp ubuntu@172.16.0.22

put db1.sql
--------------------------------
# back to orginal window
--------------------------------
UNLOCK TABLES;
QUIT;
```

# Configure Slave Server 

## repeat [ Install mysql-server and mysql-client , Configure root user, To Connet to mysql without entering passwd each time]


### Check slave user connection
```
telnet 172.16.0.11 3306

mysql -u slave -p -h 172.16.0.11
``` 

### Create Databse to replicate
```
mysql -e "create database db1"
```

### Copy database transfered from the master to slave over sftp
```
mysql -u root -p db1 < /home/ubuntu/db1.sql
```

### Edit Config of slave server
```
vim /etc/mysql/mysqld.conf.d/mysqld.cnf
---------------------------------------
[mysqld]
server-id=2
relicate-wild-do-table=db1.%      # if you want all tables on db1 to be replicated
```

### Start Slave database
```
mysql
--------
change master to master_host='172.16.0.11', master_user='slave',master_password='slave_passwd',master_log_file='mysql-bin.000001', master_log_pos=100*;

# replace master_log_file='mysql-bin.000001', master_log_pos=100* values you saved early

start slave;

show slave status\G;
```
![](https://github.com/GimhanDissanayake/mysql-master_slave_replication/blob/main/master%20slave%20replication.PNG)
### Update the master server and check the replication
