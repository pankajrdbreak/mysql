# Mysql Full/Incremental backup using percona xtrabackup tool

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
