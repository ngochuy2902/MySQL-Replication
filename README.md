## ❯ Overview
Replication enables data from one MySQL database server (known as a source) to be copied to one or more MySQL database servers (known as replicas). Replication is asynchronous by default; replicas do not need to be connected permanently to receive updates from a source. Depending on the configuration, you can replicate all databases, selected databases, or even selected tables within a database.

Advantages of replication in MySQL include:
- Scale-out solutions
- Data security
- Analytics
- Long-distance data distribution

## ❯ Scripts and Tasks
In this project, we use Docker to create 1 master server and 2 slave server

### 1. Create a network
```
docker network create replica-net
```

### 2. Create master server
```
docker run -d --name=mysql-master --net=replica-net --hostname=master-host -p 3308:3306  -e MYSQL_ROOT_PASSWORD=r00t mysql:latest --server-id=1 --log-bin='mysql-bin-1.log'
```

use `docker ps` to check if the master has already started

### 3. Create slave servers
```
docker run -d --name=mysql-slave1 --net=replica-net --hostname=slave1-host -p 3309:3306  -e MYSQL_ROOT_PASSWORD=r00t mysql:latest --server-id=2
```

```
docker run -d --name=mysql-slave2 --net=replica-net --hostname=slave2-host -p 3310:3306  -e MYSQL_ROOT_PASSWORD=r00t mysql:latest --server-id=3
```

use `docker ps` to check if the slaves have already started


### 4. Set up semi-synchronous replication (optional)
Semi-synchronous is intended to minimize data lag during data synchronization between master and slave

- Set up on master
```
docker exec -it mysql-master mysql -uroot -pr00t 
-e "INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';" 
-e "SET GLOBAL rpl_semi_sync_master_enabled = 1;" 
-e "SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 2;" 
-e "SHOW VARIABLES LIKE 'rpl_semi_sync%';"
```

- Setup on slaves
```
for N in 1 2
do docker exec -it mysql-slave$N mysql -uroot -pr00t 
-e "INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';" 
-e "SET GLOBAL rpl_semi_sync_slave_enabled = 1;" 
-e "SHOW VARIABLES LIKE 'rpl_semi_sync%';"
```

### 5. Create a slave user on master server
```
docker exec -it mysql-master mysql -uroot -pr00t 
-e "CREATE USER 'slaveuser'@'%' IDENTIFIED WITH sha256_password BY 'slavepass';" 
-e "GRANT REPLICATION SLAVE ON *.* TO 'slaveuser'@'%';" 
-e "SHOW MASTER STATUS;"
```

### 6. Set up to start slave
```
for N in 1 2
do docker exec -it mysql-slave$N mysql -uroot -pmypass 
   -e "CHANGE MASTER TO MASTER_HOST='master-host', MASTER_USER='slaveuser', 
   MASTER_PASSWORD='slavepass', MASTER_LOG_FILE='mysql-bin-1.000003';"

   docker exec -it mysql-slave$N mysql -uroot -pr00t -e "START SLAVE;"
done
```

Check slave replication status
```
docker exec -it mysql-slave1 mysql -uroot -pr00t -e "SHOW SLAVE STATUS\G"
```

```
docker exec -it mysql-slave2 mysql -uroot -pr00t -e "SHOW SLAVE STATUS\G"
```
If everything is correct, `Slave_IO_Running: Yes` and `Slave_SQL_Running: Yes` will be shown

### 7. Testing
- Create database and table on master
```
docker exec -it mysql-master mysql -uroot -pr00t -e "CREATE DATABASE tests; SHOW DATABASES;"
```

- Check database on slaves
```
docker exec -it mysql-slave1 mysql -uroot -pr00t -e "SHOW DATABASES;"
```
```
docker exec -it mysql-slave2 mysql -uroot -pr00t -e "SHOW DATABASES;"
```