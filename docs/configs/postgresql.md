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
CREATE USER admin WITH PASSWORD '************';
```

Give the user permissions:
```sql
ALTER USER admin WITH SUPERUSER;
```

Create Database:
```sql
CREATE database repository;
```

Grant permissions of database for admin user:
```sql
GRANT ALL PRIVILEGES ON DATABASE "repository" to admin;
```

## Configuration

### Allow access
Allow access to the webserver VMs.

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
psql -U admin -d repository -h <host ip address>
```
