# Multi-Source Replication with Docker MySQL Images
Setting up MySQL Multi-Source Replication (M1->S and M2->S) with Docker MySQL images

#### Replication enables data from one MySQL database server (the master) to be copied to one or more MySQL database servers (the slaves).

## References
https://dev.mysql.com/doc/refman/8.0/en/replication-multi-source.html

## 1. Overview

We start by creating a Docker network named **replicanet**, then we are going to pull **mysql 5.7** from Docker Hub (https://hub.docker.com/r/mysql/mysql-server/) and create a replication topology with 3 nodes (2 masters and 1 slave) in different hosts.

## 2. Pull MySQL Sever Image

To download the MySQL Community Edition image, the command is:
```
docker pull mysql/mysql-server:tag
```
If `:tag` is omitted, the latest tag is used, and the image for the `latest` GA version of MySQL Server is downloaded.

Examples:
```
$ docker pull mysql/mysql-server
$ docker pull mysql/mysql-server:5.7
$ docker pull mysql/mysql-server:8.0
```
In this example, we are going to use ***mysql/mysql-server:5.7***

## 3. Creating a Docker network
Fire the following command to create a network:
```
$ docker network create replicanet
```
You just need to create it once, unless you remove it from Docker.

To see all Docker networks:
```
$ docker network ls
```
## 4. Creating 3 MySQL containers

Run the commands below in a terminal.
```
docker run -d --rm --name=master1 --net=replicanet --hostname=master1 \
  -v $PWD/d0:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:5.7 \
  --server-id=1 \
  --log-bin='mysql-bin-1.log' \
  --relay_log_info_repository=TABLE \
  --master-info-repository=TABLE \
  --gtid-mode=on \
  --enforce-gtid-consistency

docker run -d --rm --name=master2 --net=replicanet --hostname=master2 \
  -v $PWD/d1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:5.7 \
  --server-id=2 \
  --log-bin='mysql-bin-1.log' \
  --master-info-repository=TABLE \
  --relay_log_info_repository=TABLE \
  --gtid-mode=on \
  --enforce-gtid-consistency

docker run -d --rm --name=slave --net=replicanet --hostname=slave \
  -v $PWD/d2:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:5.7 \
  --server-id=3 \
  --master-info-repository=TABLE \
  --relay_log_info_repository=TABLE \
  --gtid-mode=on \
  --enforce-gtid-consistency
```
It's possible to see whether the containers are started by running:
```
$ docker ps -a
```
```console
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS                             PORTS                 NAMES
f6611b651069        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   5 seconds ago       Up 3 seconds (health: starting)   3306/tcp, 33060/tcp   slave
ad3345191e8d        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   7 seconds ago       Up 5 seconds (health: starting)   3306/tcp, 33060/tcp   master2
4375760897d8        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   8 seconds ago       Up 6 seconds (health: starting)   3306/tcp, 33060/tcp   master1
```
Servers are still with status **(health: starting)**, wait till they are with state **(healthy)** before running the following commands.
```console
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS                    PORTS                 NAMES
f6611b651069        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   32 seconds ago      Up 31 seconds (healthy)   3306/tcp, 33060/tcp   slave
ad3345191e8d        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   34 seconds ago      Up 33 seconds (healthy)   3306/tcp, 33060/tcp   master2
4375760897d8        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   35 seconds ago      Up 34 seconds (healthy)   3306/tcp, 33060/tcp   master1
```
## 5. Configuring masters and slave
### 5.1. Master1
Now we’re ready start our instances and configure replication.

Let's configure in **master1 node** the replication user **"repl1"**.
```
docker exec -it master1 mysql -uroot -pmypass \
  -e "CREATE USER 'repl1'@'%' IDENTIFIED BY 'slavepass';" \
  -e "GRANT REPLICATION SLAVE ON *.* TO 'repl1'@'%';" \
  -e "SHOW MASTER STATUS;"
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+----------+--------------+------------------+------------------------------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                        |
+--------------------+----------+--------------+------------------+------------------------------------------+
| mysql-bin-1.000003 |      597 |              |                  | 646dea60-5dd4-11e8-b171-0242ac140002:1-2 |
+--------------------+----------+--------------+------------------+------------------------------------------+
```
### 5.2. Master2
Let's configure in **master2 node** the replication user **"repl2"**.
```
docker exec -it master2 mysql -uroot -pmypass \
  -e "CREATE USER 'repl2'@'%' IDENTIFIED BY 'slavepass';" \
  -e "GRANT REPLICATION SLAVE ON *.* TO 'repl2'@'%';" \
  -e "SHOW MASTER STATUS;"
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+----------+--------------+------------------+------------------------------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                        |
+--------------------+----------+--------------+------------------+------------------------------------------+
| mysql-bin-1.000003 |      597 |              |                  | 6543e379-5dd4-11e8-b17e-0242ac140003:1-2 |
+--------------------+----------+--------------+------------------+------------------------------------------+
```
### 5.3. Slave
Let’s continue with the **slave** instance.

M1 -> S
```
docker exec -it slave mysql -uroot -pmypass \
  -e "CHANGE MASTER TO MASTER_HOST='master1', MASTER_USER='repl1', \
    MASTER_PASSWORD='slavepass', MASTER_AUTO_POSITION = 1 \
    FOR CHANNEL 'master1';"
```
M2 -> S
```
docker exec -it slave mysql -uroot -pmypass \
  -e "CHANGE MASTER TO MASTER_HOST='master2', MASTER_USER='repl2', \
    MASTER_PASSWORD='slavepass', MASTER_AUTO_POSITION = 1 \
    FOR CHANNEL 'master2';"
```
Let's start replication and check whether it's working..
```
$ docker exec -it slave mysql -uroot -pmypass -e "START SLAVE;" -e "SHOW SLAVE STATUS\G"
```
Slave output:
```console
*************************** 1. row ***************************
               Slave_IO_State: Queueing master event to the relay log
                  Master_Host: master1
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin-1.000003
          Read_Master_Log_Pos: 595
               Relay_Log_File: slave-relay-bin-master1.000002
                Relay_Log_Pos: 203
        Relay_Master_Log_File: mysql-bin-1.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
                             ...
*************************** 2. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: master2
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin-1.000003
          Read_Master_Log_Pos: 595
               Relay_Log_File: slave-relay-bin-master2.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin-1.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
                             ...
```

You can see that both **Slave_IO_Running: Yes** and **Slave_SQL_Running: Yes** are running.

## 6. Inserting some data
Now it's time to test whether data is replicated to slave.

We are going to create a new database named "TEST1" in master1 and "TEST2" in master2.
```
$ docker exec -it master1 mysql -uroot -pmypass -e "CREATE DATABASE TEST1; SHOW DATABASES;"
```
Master1 output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| TEST1              |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```
```
$ docker exec -it master2 mysql -uroot -pmypass -e "CREATE DATABASE TEST2; SHOW DATABASES;"
```
Master2 output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| TEST2              |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

Run the code below to check whether the database was replicated.
```
docker exec -it slave mysql -uroot -pmypass \
  -e "SHOW VARIABLES WHERE Variable_name = 'hostname';" \
  -e "SHOW DATABASES;"
```
Slave output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| hostname      | slave |
+---------------+-------+
+--------------------+
| Database           |
+--------------------+
| information_schema |
| TEST1              |
| TEST2              |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

## 7. Stopping containers, removing created network and image

#### Stopping running container(s):
```
$ docker stop master1 master2 slave
```
#### Removing the data directories created (they are located in the folder were the containers were run):
```
$ sudo rm -rf d0 d1 d2
```
#### Removing the created network:
```
$ docker network rm replicanet
```
#### Removing MySQL image:
```
$ docker rmi mysql/mysql-server:5.7
```
