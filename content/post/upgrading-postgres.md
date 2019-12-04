---
title: "Upgrading a PostgreSQL Server"
date: 2019-12-04
keywords:
- Postgres
- PostgreSQL
- Upgrade Postgres
---

The following are my notes on how to upgrade a PostgreSQL server. A few things to consider:

- I run a **CentOS** machine - you should be able to follow along with most OSs and package managers, though.
- This is a **v11 to v12** upgrade - if this is not the case for you, just edit the commands.
- This upgrade strategy **will** incur on some database downtime - this should be minimal if you properly prepare.


```
$ yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
$ yum install postgresql12-server
$ yum install postgresql12-contrib (needed for some pg extensions)

$ systemctl stop postgresql-11

$ su - postgres

$ /usr/pgsql-12/bin/initdb -D /var/lib/pgsql/12/data/

// Optional: psql -> CREATE EXTENSION pg_stat_statements;

// Notice the --link flag, without this the uprade will take much longer and duplicate data across the filesystem.
$ /usr/pgsql-12/bin/pg_upgrade --old-datadir /var/lib/pgsql/11/data/ --new-datadir /var/lib/pgsql/12/data/ --old-bindir /usr/pgsql-11/bin/ --new-bindir /usr/pgsql-12/bin/ --link

// Edit /var/lib/pgsql/12/data/pg_hba.conf and copy settings from previous version.
// Edit /var/lib/pgsql/12/data/postgresql.conf and copy settings from previous version - Or visit take the opportunity to use PGTune.
// Make sure to address the following:
// listen_addresses = '*' (this will allow remote connections, so make sure you know what you are doing).
// log_timezone = 'UTC'
// timezone = 'UTC'

(switch back to root)
$ systemctl start postgresql-12.service

$ su - postgres

// Depending on the size and nature of your database, this could take quite a while to run, but at this point the database is up and running!
$ ./analyze_new_cluster.sh

(switch back to root)
$ systemctl enable postgresql-12

$ su - postgres
$ ./delete_old_cluster.sh
```