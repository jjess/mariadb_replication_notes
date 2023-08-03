# References

How-to: Setup a MariaDB Cluster with Galera and HAProxy - Cyral
https://cyral.com/blog/how-to-galera-mariadb-haproxy/

Setting up a Replica with Mariabackup - MariaDB Knowledge Base
https://mariadb.com/kb/en/setting-up-a-replica-with-mariabackup/

How to set up database replication with MariaDB | TechRepublic
https://www.techrepublic.com/article/how-to-set-up-database-replication-with-mariadb/

MariaDB Replication: 2 Easy Methods
https://hevodata.com/learn/mariadb-replication-easy-steps/


Migrating MySQL Databases with No Downtime - For Non-DBAs - Autoize
https://autoize.com/migrating-mysql-databases-with-no-downtime-for-non-dbas/



# test1

## How-to: Setup a MariaDB Cluster with Galera and HAProxy - Cyral

https://cyral.com/blog/how-to-galera-mariadb-haproxy/

https://www.digitalocean.com/community/tutorials/how-to-configure-a-galera-cluster-with-mariadb-on-centos-7-servers

https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=259795


Si no existe el script 'galera_new_cluster' hay que arrancar mysql con opciones.
En FreeBSD /etc/rc.conf:


mysql_enable="YES"
# cluster
# this is only necesary for the first node in cluster
mysql_args="--wsrep-new-cluster"


# test2

Parece que un cluster de galera tiene problemas cuando se quieren hacer
cambios de schema, porque se bloquea todo el cluster durante el Alter.
Hay soluciones con RSU (rolling schema update) pero es todo mÃ¡s complicado

Una alternativa es la replicaciÃ³n MASTER-MASTER o MASTER-SLAVE.

Vamos a probar:

MASTER galera-mariadb-1
    |
    +-> MASTER galera-mariadb-2
    +-> MASTER galera-mariadb-3

https://woshub.com/configure-mariadb-replication/


## configuraciÃ³n en el nodo master1

En el nodo master1 (galera-mariadb-1) aÃ±adimos configuraciÃ³n:

/usr/local/etc/mysql/conf.d/replication.cnf

```
[mysqld]
server_id = 1
report_host = master1
log_bin = /var/db/mysql/mariadb-bin
log_bin_index = /var/db/mysql/mariadb-bin.index
relay_log = /var/db/mysql/relay-bin
relay_log_index = /var/db/mysql/relay-bin.index
```

Reiniciar mariadb.


En el master1 creamos un usuario con permisos de replication:

```
create user 'replica_master'@'%' identified by 'replica_master';
grant replication slave on *.* to 'replica_master'@'%';
flush privileges;
```

Ver el fichero y posiciÃ³n para despuÃ©s:

```
root@localhost [(none)]> show master status
    -> ;
+--------------------+----------+--------------+------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+--------------------+----------+--------------+------------------+
| mariadb-bin.000001 |     1238 |              |                  |
+--------------------+----------+--------------+------------------+
1 row in set (0.000 sec)

```


## ConfiguraciÃ³n del segundo master2:

/usr/local/etc/mysql/conf.d/replication.cnf

```
[mysqld]
server_id = 2
report_host = master2
log_bin = /var/db/mysql/mariadb-bin
log_bin_index = /var/db/mysql/mariadb-bin.index
relay_log = /var/db/mysql/relay-bin
relay_log_index = /var/db/mysql/relay-bin.index
```

```
create user 'replica_master2'@'%' identified by 'replica_master';
grant replication slave on *.* to 'replica_master2'@'%';
flush privileges;
```




En master2 (galera-mariadb-2):

```
stop slave
```


```
CHANGE MASTER TO MASTER_HOST='10.0.0.4', 
       MASTER_USER='replica_master', 
       MASTER_PASSWORD='replica_master', 
       MASTER_LOG_FILE='mariadb-bin.000001',
       MASTER_LOG_POS=1238;
```

Importante, poner el Position obtenido en el master1 cuando se metiÃ³ 
el comando 'SHOW MASTER STATUS', en este caso 786


```
start slave
```

Ver fichero y posiciÃ³n:

```
root@localhost [(none)]> show master status;
+--------------------+----------+--------------+------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+--------------------+----------+--------------+------------------+
| mariadb-bin.000001 |      788 |              |                  |
+--------------------+----------+--------------+------------------+
1 row in set (0.000 sec)

```


## vuelta al master1


```
stop slave
```


```
CHANGE MASTER TO MASTER_HOST='10.0.0.5', 
       MASTER_USER='replica_master2', 
       MASTER_PASSWORD='replica_master', 
       MASTER_LOG_FILE='mariadb-bin.000001',
       MASTER_LOG_POS=788;
```

Importante, poner el Position obtenido en el master1 cuando se metiÃ³ 
el comando 'SHOW MASTER STATUS', en este caso 786


```
start slave
```

## verificar status

En el master1 (galera-mariadb-1) :

```
root@localhost [(none)]> show slave status \G ;
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 10.0.0.5
                   Master_User: replica_master2
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: mariadb-bin.000001
           Read_Master_Log_Pos: 788
                Relay_Log_File: relay-bin.000002
                 Relay_Log_Pos: 557
         Relay_Master_Log_File: mariadb-bin.000001
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes
               Replicate_Do_DB: 
           Replicate_Ignore_DB: 
            Replicate_Do_Table: 
        Replicate_Ignore_Table: 
       Replicate_Wild_Do_Table: 
   Replicate_Wild_Ignore_Table: 
                    Last_Errno: 0
                    Last_Error: 
                  Skip_Counter: 0
           Exec_Master_Log_Pos: 788
               Relay_Log_Space: 860
               Until_Condition: None
                Until_Log_File: 
                 Until_Log_Pos: 0
            Master_SSL_Allowed: No
            Master_SSL_CA_File: 
            Master_SSL_CA_Path: 
               Master_SSL_Cert: 
             Master_SSL_Cipher: 
                Master_SSL_Key: 
         Seconds_Behind_Master: 0
 Master_SSL_Verify_Server_Cert: No
                 Last_IO_Errno: 0
                 Last_IO_Error: 
                Last_SQL_Errno: 0
                Last_SQL_Error: 
   Replicate_Ignore_Server_Ids: 
              Master_Server_Id: 2
                Master_SSL_Crl: 
            Master_SSL_Crlpath: 
                    Using_Gtid: No
                   Gtid_IO_Pos: 
       Replicate_Do_Domain_Ids: 
   Replicate_Ignore_Domain_Ids: 
                 Parallel_Mode: optimistic
                     SQL_Delay: 0
           SQL_Remaining_Delay: NULL
       Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
              Slave_DDL_Groups: 0
Slave_Non_Transactional_Groups: 0
    Slave_Transactional_Groups: 0
1 row in set (0.000 sec)
```


En el master2 (galera-mariadb-2) :


```
root@localhost [(none)]> show slave status \G;
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 10.0.0.4
                   Master_User: replica_master
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: mariadb-bin.000001
           Read_Master_Log_Pos: 1238
                Relay_Log_File: relay-bin.000002
                 Relay_Log_Pos: 557
         Relay_Master_Log_File: mariadb-bin.000001
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes
               Replicate_Do_DB: 
           Replicate_Ignore_DB: 
            Replicate_Do_Table: 
        Replicate_Ignore_Table: 
       Replicate_Wild_Do_Table: 
   Replicate_Wild_Ignore_Table: 
                    Last_Errno: 0
                    Last_Error: 
                  Skip_Counter: 0
           Exec_Master_Log_Pos: 1238
               Relay_Log_Space: 860
               Until_Condition: None
                Until_Log_File: 
                 Until_Log_Pos: 0
            Master_SSL_Allowed: No
            Master_SSL_CA_File: 
            Master_SSL_CA_Path: 
               Master_SSL_Cert: 
             Master_SSL_Cipher: 
                Master_SSL_Key: 
         Seconds_Behind_Master: 0
 Master_SSL_Verify_Server_Cert: No
                 Last_IO_Errno: 0
                 Last_IO_Error: 
                Last_SQL_Errno: 0
                Last_SQL_Error: 
   Replicate_Ignore_Server_Ids: 
              Master_Server_Id: 1
                Master_SSL_Crl: 
            Master_SSL_Crlpath: 
                    Using_Gtid: No
                   Gtid_IO_Pos: 
       Replicate_Do_Domain_Ids: 
   Replicate_Ignore_Domain_Ids: 
                 Parallel_Mode: optimistic
                     SQL_Delay: 0
           SQL_Remaining_Delay: NULL
       Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
              Slave_DDL_Groups: 0
Slave_Non_Transactional_Groups: 0
    Slave_Transactional_Groups: 0
1 row in set (0.000 sec)

```

Hay que ver que  no hay errores y sobre todo:

```
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 10.0.0.4
                   Master_User: replica_master
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: mariadb-bin.000001
           Read_Master_Log_Pos: 1238
                Relay_Log_File: relay-bin.000002
                 Relay_Log_Pos: 557
         Relay_Master_Log_File: mariadb-bin.000001
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes

```


En el master1 creamos una base de datos:

```

-- --------------------------------------------------------
-- Host:                         10.0.0.4
-- Server version:               10.6.14-MariaDB-log - FreeBSD Ports
-- Server OS:                    FreeBSD13.1
-- HeidiSQL Version:             12.5.0.6677
-- --------------------------------------------------------

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET NAMES utf8 */;
/*!50503 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;


-- Dumping database structure for prueba
CREATE DATABASE IF NOT EXISTS `prueba` /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci */;
USE `prueba`;

-- Dumping structure for table prueba.tabla1
CREATE TABLE IF NOT EXISTS `tabla1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- Dumping data for table prueba.tabla1: ~0 rows (approximately)

/*!40103 SET TIME_ZONE=IFNULL(@OLD_TIME_ZONE, 'system') */;
/*!40101 SET SQL_MODE=IFNULL(@OLD_SQL_MODE, '') */;
/*!40014 SET FOREIGN_KEY_CHECKS=IFNULL(@OLD_FOREIGN_KEY_CHECKS, 1) */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40111 SET SQL_NOTES=IFNULL(@OLD_SQL_NOTES, 1) */;

```

En master2 se ve la misma base de datos y tabla.

En master2 creamos un registro nuevo en la tabla:

```
INSERT INTO `prueba`.`tabla1` (`name`) VALUES ('pepe');
```

En master1:

```
root@localhost [prueba]> select * from tabla1;
+----+------+
| id | name |
+----+------+
|  1 | pepe |
+----+------+
1 row in set (0.001 sec)

```

Creamos una tabla en master2:

```
CREATE TABLE `tabla2` (
	`id` INT NOT NULL AUTO_INCREMENT,
	`name2` VARCHAR(50) NOT NULL DEFAULT '0',
	PRIMARY KEY (`id`)
)
COLLATE='utf8mb4_general_ci'
;
```

En master1:

```
root@localhost [prueba]> describe tabla2;
+-------+-------------+------+-----+---------+----------------+
| Field | Type        | Null | Key | Default | Extra          |
+-------+-------------+------+-----+---------+----------------+
| id    | int(11)     | NO   | PRI | NULL    | auto_increment |
| name2 | varchar(50) | NO   |     | 0       |                |
+-------+-------------+------+-----+---------+----------------+
2 rows in set (0.002 sec)

```

En principio todo funciona.



Un documento interesante :

https://www.howtoforge.com/how-to-setup-mariadb-master-master-replication-on-debian-11/


Otra cosa interesante:

https://mariadb.com/kb/en/setting-up-replication/
    HabrÃ­a que activar:
        
        Use Global Transaction Id (GTID)
        MariaDB starting with 10.0

        MariaDB 10.0 introduced global transaction IDs (GTIDs) for replication. It is generally recommended to use (GTIDs) from MariaDB 10.0, as this has a number of benefits. All that is needed is to add the MASTER_USE_GTID option to the CHANGE MASTER statement, for example:

        CHANGE MASTER TO MASTER_USE_GTID = slave_pos


https://mariadb.com/kb/en/setting-up-a-replica-with-mariabackup/



Para evitar el problema de split-brain quizÃ¡ convenga mÃ¡s el modelo master-slave



Un caso de uso de replicas con HA-Proxy:

https://stackoverflow.com/questions/28089708/haproxy-for-mysql-master-slave-replication

Un artÃ­culo sobre migrar una base de datos y llevarse una base de datos grande y ademÃ¡s con SSL:

https://autoize.com/migrating-mysql-databases-with-no-downtime-for-non-dbas/




# Mover datos a otro servidor con rsync (en lugar de mysqldump)

https://www.looklinux.com/how-to-setup-mysql-master-slave-replication-using-rsync/

