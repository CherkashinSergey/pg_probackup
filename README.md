# pg_probackup

`pg_probackup` is a utility to manage backup and recovery of PostgreSQL database clusters. It is designed to perform periodic backups of the PostgreSQL instance that enable you to restore the server in case of a failure.

The utility is compatible with:
* PostgreSQL 9.5, 9.6, 10, 11;

`PTRACK` backup support provided via following options:
* vanilla PostgreSQL compiled with ptrack patch. Currently there are patches for [PostgreSQL 9.6](https://gist.githubusercontent.com/gsmol/5b615c971dfd461c76ef41a118ff4d97/raw/e471251983f14e980041f43bea7709b8246f4178/ptrack_9.6.6_v1.5.patch) and [PostgreSQL 10](https://gist.githubusercontent.com/gsmol/be8ee2a132b88463821021fd910d960e/raw/de24f9499f4f314a4a3e5fae5ed4edb945964df8/ptrack_10.1_v1.5.patch)
* Postgres Pro Standard 9.5, 9.6, 10
* Postgres Pro Enterprise 9.5, 9.6, 10

As compared to other backup solutions, `pg_probackup` offers the following benefits that can help you implement different backup strategies and deal with large amounts of data:
* Choosing between full and page-level incremental backups to speed up backup and recovery
* Implementing a single backup strategy for multi-server PostgreSQL clusters
* Automatic data consistency checks and on-demand backup validation without actual data recovery
* Managing backups in accordance with retention policy
* Running backup, restore, and validation processes on multiple parallel threads
* Storing backup data in a compressed state to save disk space
* Taking backups from a standby server to avoid extra load on the master server
* Extended logging settings
* Custom commands to simplify WAL log archiving

To manage backup data, `pg_probackup` creates a backup catalog. This directory stores all backup files with additional meta information, as well as WAL archives required for [point-in-time recovery](https://postgrespro.com/docs/postgresql/current/continuous-archiving.html). You can store backups for different instances in separate subdirectories of a single backup catalog.

Using `pg_probackup`, you can take full or incremental backups:
* `Full` backups contain all the data files required to restore the database cluster from scratch.
* `Incremental` backups only store the data that has changed since the previous backup. It allows to decrease the backup size and speed up backup operations. `pg_probackup` supports the following modes of incremental backups:
  * `PAGE` backup. In this mode, `pg_probackup` scans all WAL files in the archive from the moment the previous full or incremental backup was taken. Newly created backups contain only the pages that were mentioned in WAL records. This requires all the WAL files since the previous backup to be present in the WAL archive. If the size of these files is comparable to the total size of the database cluster files, speedup is smaller, but the backup still takes less space.
  * `DELTA` backup. In this mode, `pg_probackup` read all data files in PGDATA directory and only those pages, that where changed since previous backup, are copied. Continuous archiving is not necessary for it to operate. Also this mode could impose read-only I/O pressure equal to `Full` backup.
  * `PTRACK` backup. In this mode, PostgreSQL tracks page changes on the fly. Continuous archiving is not necessary for it to operate. Each time a relation page is updated, this page is marked in a special `PTRACK` bitmap for this relation. As one page requires just one bit in the `PTRACK` fork, such bitmaps are quite small. Tracking implies some minor overhead on the database server operation, but speeds up incremental backups significantly.

Regardless of the chosen backup type, all backups taken with `pg_probackup` support the following archiving strategies:
* `Autonomous backups` include all the files required to restore the cluster to a consistent state at the time the backup was taken. Even if continuous archiving is not set up, the required WAL segments are included into the backup.
* `Archive backups` rely on continuous archiving. Such backups enable cluster recovery to an arbitrary point after the backup was taken (point-in-time recovery).

## Limitations

`pg_probackup` currently has the following limitations:
* Creating backups from a remote server is currently not supported.
* The server from which the backup was taken and the restored server must be compatible by the [block_size](https://postgrespro.com/docs/postgresql/current/runtime-config-preset#guc-block-size) and [wal_block_size](https://postgrespro.com/docs/postgresql/current/runtime-config-preset#guc-wal-block-size) parameters and have the same major release number.
* Microsoft Windows operating system is not supported.
* Configuration files outside of PostgreSQL data directory are not included into the backup and should be backed up separately.

## Installation and Setup
### Linux Installation
```shell
#DEB Ubuntu|Debian Packages
echo "deb [arch=amd64] http://repo.postgrespro.ru/pg_probackup/deb/ $(lsb_release -cs) main-$(lsb_release -cs)" > /etc/apt/sources.list.d/pg_probackup.list
wget -O - http://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG_PROBACKUP | apt-key add - && apt-get update
apt-get install pg-probackup-{10,9.6,9.5}

#DEB-SRC Packages
echo "deb-src [arch=amd64] http://repo.postgrespro.ru/pg_probackup/deb/ $(lsb_release -cs) main-$(lsb_release -cs)" >>\
  /etc/apt/sources.list.d/pg_probackup.list
apt-get source pg-probackup-{10,9.6,9.5}

#RPM Centos Packages
rpm -ivh http://repo.postgrespro.ru/pg_probackup/keys/pg_probackup-repo-centos.noarch.rpm
yum install pg_probackup-{10,9.6,9.5}

#RPM RHEL Packages
rpm -ivh http://repo.postgrespro.ru/pg_probackup/keys/pg_probackup-repo-rhel.noarch.rpm
yum install pg_probackup-{10,9.6,9.5}

#RPM Oracle Linux Packages
rpm -ivh http://repo.postgrespro.ru/pg_probackup/keys/pg_probackup-repo-oraclelinux.noarch.rpm
yum install pg_probackup-{10,9.6,9.5}

#SRPM Packages
yumdownloader --source pg_probackup-{10,9.6,9.5}
```

To compile `pg_probackup`, you must have a PostgreSQL installation and raw source tree. To install `pg_probackup`, execute this in the module's directory:

```shell
make USE_PGXS=1 PG_CONFIG=<path_to_pg_config> top_srcdir=<path_to_PostgreSQL_source_tree>
```

Once you have `pg_probackup` installed, complete [the setup](https://postgrespro.com/docs/postgrespro/current/app-pgprobackup.html#pg-probackup-install-and-setup).

## Documentation

Currently the latest documentation can be found at [Postgres Pro Enterprise documentation](https://postgrespro.com/docs/postgrespro/current/app-pgprobackup).

## Licence

This module available under the same license as [PostgreSQL](https://www.postgresql.org/about/licence/).

## Feedback

Do not hesitate to post your issues, questions and new ideas at the [issues](https://github.com/postgrespro/pg_probackup/issues) page.

## Authors

Postgres Professional, Moscow, Russia.

## Credits

`pg_probackup` utility is based on `pg_arman`, that was originally written by NTT and then developed and maintained by Michael Paquier.
