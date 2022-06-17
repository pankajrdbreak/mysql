# Mysql Full/Incremental backup using percona xtrabackup tool

Prequisite: Mysql Server with test db

1. First we will take full backup of database
We have script for that we will run it 

```console
root@node:~# sh finalbkp.sh full
root@node:~# cd /data/backup/2022-06-17/base/
root@node:/data/backup/2022-06-17/base# ll
total 20552
drwxr-x--- 11 root root     4096 Jun 17 12:20 ./
drwxr-xr-x  3 root root     4096 Jun 17 12:19 ../
-rw-r-----  1 root root      487 Jun 17 12:19 backup-my.cnf
drwxr-x---  2 root root     4096 Jun 17 12:19 .cache/
drwxr-x---  2 root root     4096 Jun 17 12:19 .config/
drwxr-x---  2 root root     4096 Jun 17 12:20 emp/
-rw-r-----  1 root root      292 Jun 17 12:19 ib_buffer_pool
-rw-r-----  1 root root 12582912 Jun 17 12:20 ibdata1
drwxr-x---  2 root root     4096 Jun 17 12:19 .local/
drwxr-x---  2 root root     4096 Jun 17 12:20 mysql/
drwxr-x---  2 root root     4096 Jun 17 12:20 PERCONA_SCHEMA/
drwxr-x---  2 root root     4096 Jun 17 12:19 performance_schema/
drwxr-x---  2 root root    12288 Jun 17 12:20 sys/
drwxr-x---  2 root root     4096 Jun 17 12:20 test/
-rw-r-----  1 root root      133 Jun 17 12:20 xtrabackup_checkpoints
-rw-r-----  1 root root      458 Jun 17 12:19 xtrabackup_info
-rw-r-----  1 root root  8388608 Jun 17 12:20 xtrabackup_logfile
-rw-r--r--  1 root root        1 Jun 17 12:20 xtrabackup_master_key_id
```
It will create above above of folders and files

2. Now we will have to prepare this backup as consistent to restore 
```console
root@node:~# xtrabackup --prepare --apply-log-only --export --target-dir=/data/backup/2022-06-17/base
```
Here --export option is used so that we can restore single table also

3. Now to restore you can delete the entire mysql directory folders and files
```console 
root@node:~# systemctl stop mysql
root@node:~# rm -rf /var/lib/mysql/*
root@node:~# xtrabackup --copy-back --target-dir=/data/backup/base/
root@node:~# chown -R mysql. /var/lib/mysql
root@node:~# systemctl start mysql
```
4. All the data is restored

5. Now we will take incremental backups using full backup as a base for that
6. Also we will prepare incremental backup and attach it to full backup to restore
```console
//root@node:~# xtrabackup --backup --target-dir=/data/backup/2022-06-17/inc1 --incremental-basedir=/data/backup/2022-06-17/base/ --datadir=/var/lib/mysql
root@node:~# sh finalbkp.sh incremental
root@node:~# xtrabackup --prepare --apply-log-only --export --target-dir=/data/backup/2022-06-17/base --incremental-dir=/data/backup/2022-06-17/inc1/
root@node:~# systemctl stop mysql
root@node:~# rm -rf /var/lib/mysql/*
root@node:~# xtrabackup --copy-back --target-dir=/data/backup/base/
root@node:~# chown -R mysql. /var/lib/mysql
root@node:~# systemctl start mysql
```

7. Same process will follow for next incremental backup
```console
//root@node:~# xtrabackup --backup --target-dir=/data/backup/2022-06-17/inc2 --incremental-basedir=/data/backup/2022-06-17/base/ --datadir=/var/lib/mysql
root@node:~# sh finalbkp.sh incremental
root@node:~# xtrabackup --prepare --apply-log-only --export --target-dir=/data/backup/2022-06-17/base --incremental-dir=/data/backup/2022-06-17/inc2/
root@node:~# systemctl stop mysql
root@node:~# rm -rf /var/lib/mysql/*
root@node:~# xtrabackup --copy-back --target-dir=/data/backup/base/
root@node:~# chown -R mysql. /var/lib/mysql
root@node:~# systemctl start mysql
```

8. We should also have backups of table structures in case table is dropped
```console
root@node:~# mysqldump --no-data -h localhost -u root -ppass test cab > cab_backup.sql
```

# Situations
1) Truncate Table / Delete table data
2) Drop Table
3) Alter Table
4) Drop Database
5) Alter Database
6) Full Destroy

## In the above situations how we can perform the restore lets see...

1. If we truncate the table by mistake then we can simply restore it using below steps withour restarting server
```console
mysql> ALTER TABLE test.cab DISCARD TABLESPACE;
root@node:~# cp /data/backup/test.* /var/lib/mysql/test/
root@node:~# chown mysql.mysql /var/lib/mysql/test/
mysql> ALTER TABLE test.cab IMPORT TABLESPACE;
```
2. If we dropped table by mistake then first we have to create that table using backup structure file created in 8th step
```console
root@node:~# mysql -u root -p test < cab_backup.sql
```
Then copy .cfg .ibd .exp files to /var/lib/mysql/test/
```console
mysql> ALTER TABLE test.cab DISCARD TABLESPACE;
root@node:~# cp /data/backup/2022-06-17/base/test/cab.cfg /var/lib/mysql/test/
root@node:~# cp /data/backup/2022-06-17/base/test/cab.exp /var/lib/mysql/test/
root@node:~# cp /data/backup/2022-06-17/base/test/cab.ibd /var/lib/mysql/test/
root@node:~# chown -R mysql. /var/lib/mysql/test/
mysql> ALTER TABLE test.cab IMPORT TABLESPACE;
```
Done particular table is restored




























##rough work
```console
xtrabackup --backup --target-dir=/data/backup/base --datadir=/var/lib/mysql
xtrabackup --backup --target-dir=/data/backup/inc1 --incremental-basedir=/data/backup/base/ --datadir=/var/lib/mysql
xtrabackup --prepare --apply-log-only --target-dir=/data/backup/base
xtrabackup --prepare --apply-log-only --target-dir=/data/backup/base --incremental-dir=/data/backup/inc1/
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/data/backup/base/
chown -R mysql. /var/lib/mysql
systemctl start mysql
```

2nd increment
```console
xtrabackup --backup --target-dir=/data/backup/inc2 --incremental-basedir=/data/backup/base --datadir=/var/lib/mysql
xtrabackup --prepare --apply-log-only --target-dir=/data/backup/base --incremental-dir=/data/backup/inc2/
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/data/backup/base/
chown -R mysql. /var/lib/mysql
systemctl start mysql
```

3rd increment
```console
xtrabackup --backup --target-dir=/data/backup/inc3 --incremental-basedir=/data/backup/base --datadir=/var/lib/mysql
xtrabackup --prepare --apply-log-only --target-dir=/data/backup/base --incremental-dir=/data/backup/inc3/
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/data/backup/base/
chown -R mysql. /var/lib/mysql
systemctl start mysql
```


To restore single table which was dropped you can do fllowing

https://severalnines.com/database-blog/how-restore-single-table-using-percona-xtrabackup

This is useful when you by mistakely deleted data from the table 
Note: not useful if table is dropped

```console
mysql> ALTER TABLE test.cab DISCARD TABLESPACE;
cp /data/backup/test.* /var/lib/mysql/test/
chown mysql.mysql /var/lib/mysql/test/
mysql> ALTER TABLE test.cab IMPORT TABLESPACE;
```
