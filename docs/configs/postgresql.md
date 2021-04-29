# PostgreSQL

PostgreSQL also known as Postgres, is a free and open-source relational database management system (RDBMS) emphasizing extensibility and SQL compliance.

In this guideline we will take a look on the Postgresql database setup and configurations for [Repository](https://repository.tugraz.at) Production instance. Using PostgreSQL Version ```11```, and VM ```invenio03-prod.tugraz.at```

## Installation

```bash
$ apt-get update
$ apt-get install postgresql postgresql-contrib
```

By default PostgreSQL has a default user ```postgres```. We can access the database with this user ```su postgres``` and access the shell with  ```psql```.

To change the ```postgres``` user password:
```bash
$ su postgres
(postgres) $ psql
(postgres) ALTER USER postgres WITH PASSWORD '************';
```

## Users and Database

Create a admin user:
```sql
CREATE USER <USERNAME> WITH PASSWORD '************';
```

Give the user permissions:
```sql
ALTER USER "USERNAME" WITH SUPERUSER;
```

Create Database:
```sql
CREATE database DATABASE_NAME;
```

Grant permissions of database for created user:
```sql
GRANT ALL PRIVILEGES ON DATABASE "DATABASE_NAME" to USER;
```

## Configuration

### Allow access
Allow remote connection to the webserver VMs.

Open the ```/etc/postgresql/11/main/pg_hba.conf``` file and edit it as below:

```conf
# TYPE  DATABASE        USER         ADDRESS              METHOD
host    all             <user>       ********/28          trust
host    all             <user>       ********/28          trust
host    all             <user>       repository.tugraz.at trust
```

### Listen address
Listen to other VMs.

Open the ```/etc/postgresql/11/main/postgresql.conf``` file and edit it as below:

```diff
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

- #listen_addresses = 'localhost'         # what IP address(es) to listen on;
+ listen_addresses = '<ip addresses>'     # what IP address(es) to listen on;
```

### Test connection
Access the database from one of the configured machines.

!!! note "psql"
     Make sure you have postgres client installed.


```bash
psql -U USERNAME -d DATABASE -h HOST_IP
```

## Backup & Restore

### Requirements

* A server running Linux operating system with PostgreSQL installed.
* A root password is setup on your server.

### Backup a Single PostgreSQL Database
You will need to use pg_dump tool to backup a PostgreSQL database. This tool will dump all content of a database into a single file.

The basic syntax to backup a PostgreSQL database is shown below:

```sql
pg_dump -U [option] [database_name] > [backup_name]
```

A brief explanation of all available option is shown below:

* -U : Specify the PostgreSQL username.
* -W : Force pg_dump command to ask for a password.
* -F : Specify the format of the output file.
* -f : Specify the output file.
* p : Plain text SQL script.
* c : Specify the custom formate.
* d : Specify the directory format.
* t : Specify tar format archive file.

For example, create a backup of the PostgreSQL database named db1 in the tar format, run the following command:

```sql
pg_dump -U postgres -F c db1 > db1.tar
```

If you want to save the backup in a directory format, run the following command:

```sql
pg_dump -U postgres -F d db1 > db1_backup
```

If your database is very large and wants to generate a small backup file then you can use pg_dump with a compression tool such as gzip to compress the database backup.

```sql
pg_dump -U postgres db1 | gzip > db1.gz
```

You can also reduce the database backup time by dumping number_of_jobs tables simultaneously using the -j flag.

```sql
pg_dump -U postgres -F d -j 5 db1 -f db1_backup
```

**Note** : Also keep in mind that the above command will reduce the time of the backup but it also increases the load on the server.

### Restore a Single PostgreSQL Database
If you choose custom, directory, or archive format when taking a database backup. Then, you will need to use pg_restore command to restore your database.

The basic syntax to restore a database with pg_restore is shown below:

```sql
pg_restore -U [option] [db_name] [db_backup]
```

A brief explanation of each option is shown below:

* -c : Used to drop database objects before recreating them.
* -C : Used to create a database before restoring into it.
* -e : Exit if an error has been encountered.
* -F format : Used to specify the format of the archive.

For example, restore a backup from the file db1.tar, you will need to consider two options:

1. If the database already exists.
2. The format of your backup.

If your database already exists, you can restore it with the following command:

```sql
pg_restore -U postgres -Ft -d db1 < db1.tar
```

If your database is not exists, you can restore it with the following command:

```sql
pg_restore -U postgres -Ft -C -d db1 < db1.tar
```

### Backup a Remote PostgreSQL Database
In order to perform the database backup on the remote PostgreSQL server. You will need to configure your PostgreSQL server to allow remote connection.

The basic syntax to backup a remote PostgreSQL database is shown below:

```sql
pg_dump -h [remote-postgres-server-ip] -U [option] [database_name] > [backup_name]
```

For example, create a backup of the PostgreSQL database on the remote server ( 192.168.0.100 ) with name remote_db1 in the tar format, run the following command:

```sql
pg_dump -h 192.168.0.100 -U postgres -F c remote_db1 > remote_db1.tar
```


### Restore a Remote PostgreSQL Database

The basic syntax to restore a remote PostgreSQL database is shown below:

```sql
pg_restore -h [remote-postgres-server-ip] -U [option] [database_name] < [backup_name]
```

For example, restore a database from the file remote_db1.tar on the remote server ( 192.168.0.100 ), run the following command:

```sql
pg_restore -h 192.168.0.100 -U postgres -Ft -d remote_db1 < remote_db1.tar
```

For more information, you can see the [pg_dump](https://www.postgresql.org/docs/12/app-pgdump.html) and [pg_restore](https://www.postgresql.org/docs/12/app-pgrestore.html) reference pages.

