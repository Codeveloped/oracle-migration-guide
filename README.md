# Guide on migrating an Oracle Database

## Requirements
- LPIC-1 level of Linux knowledge, meaning the ability to perform maintenance tasks with the command line, install and configure a computer
- LPI DevOps Tools Engineer level of knowledge around Docker
- Access to the Oracle Database
- Docker
- DBeaver
- Reasonable level of sanity
- Patience

## Check which version of Oracle Database is running
Connect to the Oracle Database in DBeaver, then right click on the connection in the database navigator and click on "SQL Editor"
Then paste the following query there and execute it.
```sql
select * from v$version where banner like 'oracle%';
```
It should return data along the likes of this
```
BANNER | CON_ID

Oracle Database 12c Standard Edition Release 12.2.0.1.0 - 64bit Production | 0
PL/SQL Release 12.2.0.1.0 - Production | 0
"CORE	12.2.0.1.0	Production" | 0
TNS for Linux: Version 12.2.0.1.0 - Production | 0
NLSRTL Version 12.2.0.1.0 - Production | 0
```

The first entry shows the version number **12.2.0.1.0**. Take note of this version number, it's used throughout the next steps.  
It also shows which release candidate is being used **Standard Edition Release**. Take note of this release candidate, it's also used throughout the next steps.

## Retrieve other configuration data
TODO

## Setup local Oracle database

See [SETUP_LOCAL_ORACLE_DATABASE.md](SETUP_LOCAL_ORACLE_DATABASE.md).

TL;DR; version:

* Clone [Oracle's docker images repository](https://github.com/oracle/docker-images).
* Go to [Oracle's download page](https://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html), then download the Linux x86-64 zipped binary for the required Oracle database version.
* After the download completes, move the file to the version specific directory in  the dockerfiles directory inside the repository.
* Execute ./buildDockerimage -s -v **ORACLE_VERSION_NUMBER_HERE**, from a terminal inside the dockerfiles directory inside the repository. Replace **ORACLE_VERSION_NUMBER_HERE** with the required Oracle Database version.
* Pray the ./buildDockerImage.sh bash script supplied by Oracle doesn't break
* ???
* Profit!

## Migrate from one Oracle Database to the other
Make sure:
* The other database has been configured. See [this document](CONFIGURE_ORACLE_DATABASE.md) for instructions.
* You have shell access to both the current and new database.
* The target host where the database is being migrated to has sufficient storage.
* You posses a valid Oracle Database License that allows usage on on-premise hardware.

If you're using a cloud hosted Oracle Database, [follow these steps to get SSH access](ORACLE_CLOUD_SSH_ACCESS.md).

SSH into the instance/host running Oracle Linux and the current Oracle Database you'd like to migrate. After connecting, execute the following command:

```sh
$ sqlplus / as sysdba
```

It should greet you with text along the following lines:
```
SQL*Plus: Release 12.2.0.1.0 Production on Thu Oct 11 13:42:23 2018

Copyright (c) 1982, 2016, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Standard Edition Release 12.2.0.1.0 - 64bit Production

SQL>     
```

Input and run this query:
```sql
SELECT directory_path FROM dba_directories WHERE directory_name = 'DATA_PUMP_DIR';
```

Read the resulting path and store it somewhere for usage later. It should show something along the lines of:

```
DIRECTORY_PATH
--------------------------------------------------------------------------------
/u01/app/oracle/admin/ORCL/dpdump/
```

Now create an other directory for data pumping by executing this query:

<pre><code>CREATE DIRECTORY dpump_dir1 AS '<b>&lt;the path you read out previously&gt;</b>';
</code></pre>

Exit `sqlplus` by hitting Ctrl-D or typing `exit`. Now it's time to start the data pump export. Execute the following;
```sh
$ expdp DIRECTORY=dpump_dir1 DUMPFILE=ORACLEDATA.dmp FULL=y
```

When that process is finished, copy the created database file via `scp`.

```sh
$ scp oracle.example.com:/u01/app/oracle/admin/ORCL/dpdump/ORACLEDATA.dmp ~/ORACLEDATA.dmp
```

On the host/instance you'd like to migrate to, create an directory that will hold the database data. This will be mounted as a volume inside the Docker container.

```sh
$ sudo mkdir -p /srv/oracledb
```

Oracle's Docker container runs as user 54321 "oracle". The directory created in the previous step needs to have appropriate permissions to allow the container to access it.
```sh
$ sudo chown 54321:54321 /srv/oracledb
```

All set. Now spawn the docker container.

<pre><code>$ docker run -d --name oracledb -p 127.0.0.1:1521:1521 -p 127.0.0.1:5500:5500 \
  -e ORACLE_SID=<b>&lt;YOUR_SID&gt;</b> -e ORACLE_PDB=<b>&lt;YOUR_PDB&gt;</b> -e TZ=<b>&lt;YOUR_TIMEZONE&gt;</b> -e ORACLE_PWD=<b>&lt;YOUR_PASSWORD&gt;</b> \
  -v /srv/oracledb:/opt/oracle/oradata oracle/database:12.2.0.1-se2
</code></pre>

Observe logs by executing `docker logs -f oracledb`. If all goes well, it should print:

```
#########################
DATABASE IS READY TO USE!
#########################
```

When this is printed, you can stop watching the log files.

Now let's shell into the Docker container running Oracle database.

```sh
docker exec -it oracledb /bin/bash
```

Retrieve the data pump path by executing the following query via `sqlplus`. When asked for a user, enter `/ as sysdba`
```
SELECT directory_path FROM dba_directories WHERE directory_name = 'DATA_PUMP_DIR';
```

Now exit `sqlplus` and the container by hitting Ctrl-D or typing `exit`.

Set the right permissions on the data dump, so the container can access it.
```sh
$ sudo chown 54321:54321 ORACLEDATA.dmp
```

Copy over the dump to the container:
<pre><code>$ sudo docker cp ~/ORACLEDATA.dmp oracledb:<b>&lt;path from previous step&gt;</b>/ORACLEDATA.dmp
</code></pre>

If it spits out an error 'the directory does not exist', shell into the container and create the directory via `mkdir`.

Now it's time to import the data. Shell into the container, execute `sqlplus` and run the following query:

```sql
CREATE DIRECTORY dpump_dir1 AS '<path from previous step>';
```

Exit `sqlplus` and execute the following command:

```sh
impdp DIRECTORY=dpump_dir1 DUMPFILE=ORACLEDATA.dmp
```

When asked for user credentials, enter `/ as sysdba`.

Should be all set after that.