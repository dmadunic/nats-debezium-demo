# NATS debezium Demo
This project demonstrates how to setup PostgreSQL, debezium and NATS.

The goal is to setup debezium to capture and data changes in filtered set of tables located in different schemas, and to publish those changes in local NATS.


## (Local) Postgres setup
**PREREQUISITES:** 
- Installed and configured local Postgres instance
- User with non admin priviliges
- User with database admin priviliges
- Installed docker
- nats cli installed

### 1. Create 'test' database
Connect to Postgres default database (postgres) with user with administrative priviliges and execute:

```sql
CREATE DATABASE test;
```
Grant access to this database to existing non admin user

```sql
GRANT ALL PRIVILEGES ON DATABASE test TO domagoj;
```
This will grant CONNECT and CREATE priviliges to this user.

### 2. Create table in public schema owned by non admin user
Connect to test database as non admin user adn execute:
```sql
-- dumy table in public schema owned by domagoj ---
create table profile (id serial primary key, name text, color text);

insert into profile (name, color) values ('Joe', 'blue');
insert into profile (name, color) values ('Pam', 'green');
```

### 3. Create geodata schema and user
Connect to test database with administrative user and execute:

```sql
CREATE ROLE geodata NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT LOGIN PASSWORD 'geodatapwd';
-- setup geodata user ---
GRANT ALL PRIVILEGES ON DATABASE test TO geodata;
CREATE SCHEMA IF NOT EXISTS AUTHORIZATION "geodata";
```

Now populate geodata schema by running the following command inside **geodata/** folder
```sh
docker-compose up
```
This will startup geodata application which will create all necessary tables in geodata schema and populate them with initial data.

Once the application strats up, shut it down with:

```sh
docker-compose down
```
Now you will have database with the two schemas (public and geodata) each with several tables:

```
 Schema |  Name   | Type  |         Owner          
--------+---------+-------+------------------------
 public | profile | table | domagoj
(1 row)

 Schema  |         Name          | Type  |         Owner          
---------+-----------------------+-------+------------------------
 geodata | country               | table | geodata
 geodata | currency              | table | geodata
 geodata | databasechangelog     | table | geodata
 geodata | databasechangeloglock | table | geodata
 geodata | jhi_authority         | table | geodata
 geodata | jhi_user              | table | geodata
 geodata | jhi_user_authority    | table | geodata
(7 rows)
```

## Setup replication role
Now we will configure postgres and debezium to capture changes only in the following tables:
- public.profile
- geodata.country
- geodata.currency

### 1. Create replication user and group
Connect to test database with user with administrative priviliges.

```sql
-- setup REPLICATION role ---
CREATE ROLE debezium REPLICATION LOGIN password 'debezium';

-- Grant CREATE, CONNECT priviliges on database to replication role
GRANT ALL PRIVILEGES ON DATABASE test TO debezium;

-- GRANT Replication role access to existing tables in existing schemas
grant usage on schema GEODATA to DEBEZIUM;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO debezium;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA geodata TO debezium;
ALTER DEFAULT PRIVILEGES GRANT ALL ON TABLES TO debezium;

-- CREATE Replication group role
CREATE ROLE REPLICATION_GROUP_TEST;

GRANT REPLICATION_GROUP_TEST TO geodata;
GRANT REPLICATION_GROUP_TEST TO debezium;
-- Modify role geodata so that inherits priviliges of the group role (without this it will not be able to access tables transfered to group)
ALTER ROLE geodata INHERIT;

ALTER TABLE geodata.country OWNER TO REPLICATION_GROUP_TEST;
ALTER TABLE geodata.currency OWNER TO REPLICATION_GROUP_TEST;
ALTER TABLE public.profile OWNER TO REPLICATION_GROUP_TEST;
```

## 3. Edit pg_hba.conf (optional?)
Add the following line:
```
host replication debezium 0.0.0.0/0 trust
```

## 4. Edit postgresql.conf
Add the following lines to the wal section:

After editing these files you will need to restart postgres instance, by running for eample:

```
sudo /etc/init.d/postgresql restart
```

## Runing the demo
```
docker-compose up
```

In the other terminal execute: `nats stream subjects DebeziumStream`

```
        postgres.public.profile:   3       postgres.geodata.country: 248
        postgres.geodata.currency: 20
```
