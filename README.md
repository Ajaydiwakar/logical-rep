# logical-rep

root# cd/
root /# mkdir postgress

Inside postgres create two dir for clusters as clu1 and clu2 
root/postgres# mkdir clu1 clu2

back to root and assign owner for both clusters as postgres 
root/postgres# chown postgres:postgres clu1 clu2

Now login in postgres

root# sudo su - postgres

postgres@rootadmin # EXPORT PATH=$PATH:/usr/pgsql-12/bin    ( use this only if direct cluster initdb doesnt works)

Initialize db in both clusters 
postgres@osboxes:~$ /usr/lib/postgresql/12/bin/initdb -D /postgres/clu1 -U postgres        ( incase some error shows related to permission than make sure you follow the step in line 9 and 10 

postgres@osboxes:~$ /usr/lib/postgresql/12/bin/initdb -D /postgres/clu2 -U postgres


// Publisher Side 

Now Go to - cluster 1 pg_hba.conf file
postgres@osboxes:/postgres$ vi clu1/pg_hba.conf 
 ( look for the line ipv4hosts below  and add a newline with same data only change the user as repuser in user column .)

Now next go for postgresql.conf file of clu1 
postgres@osboxes:/postgres$ vi clu1/postgresql.conf

Here: In postgresql.conf of clu1 look for port no and uncomment it and replace 5432 with 9999. Next look for wal_level   , change it to logical from replica.

Once done now start the cluster 
postgres@osboxes:/$ /usr/lib/postgresql/12/bin/pg_ctl -D /postgres/clu1 start

login into cluster 9999
postgres@osboxes:/$ psql -p 9999
postgres=# \l      ( no db found)

postgres=# CREATE DATABASE pub;
postgres=# \l    ( you will see database pub listed here)

connect db pub
postgres=# \c pub

next create a table and fetch some data in it 

pub=# CREATE TABLE t1(a int primary key , b int);
CREATE TABLE
pub=# INSERT INTO t1 values(1,10);
INSERT 0 1
pub=# INSERT INTO t1 values(2,20);
INSERT 0 1
pub=# \d
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

pub=# select *from t1;
 a | b  
---+----
 1 | 10
 2 | 20
(2 rows)

exit and relogin  as ROLE has to be created outside the db conenction 
pub=# \q

postgres@osboxes:/$ psql -p 9999

postgres=# CREATE ROLE repuser REPLICATION LOGIN PASSWORD 'password';
CREATE ROLE

conenct db pub now 
postgres=# \c pub

Now inside db pub we are going to create PUBLICATION for all tables
pub=# CREATE PUBLICATION my_pub FOR ALL TABLES;
CREATE PUBLICATION

Next need to GRANT permission to table  to repuser
pub=# GRANT SELECT ON t1 TO repuser;
GRANT

Now inorder to fetch data of this table directly to subscriber db we can use pg_dump
postgres@osboxes:/$ pg_dump -t t1 -s pub -p 9999;        ( here 9999 is your publisher port no)

postgres@osboxes:/$ pg_dump -t t1 -s pub -p 9999 | psql -p 8888 sub    ( this will fetch all table data to subscriber sub db )

You are now connected to database "pub" as user "postgres".
pub=# INSERT INTO t1 values(25, 1000);
INSERT 0 1
pub=# UPDATE t1 set a=29 where b=20; 
UPDATE 1
pub=# TRUNCATE table t1;


// Subscriber Side 

login in cluster port 8888
postgres@osboxes:~$ psql -p 8888
postgres=# \l

Now create a db named sub 
postgres=# create database sub;
CREATE DATABASE

postgres=# \l
Yo will see your db listed here in db lists

connect db sub and execute \d
postgres=# \c sub;
sub=# \d

You will see that a table named t1 will be shown , same table t1 was created in publisher side

Use this command to check details of table t1 type
sub=# \d+ t1

NOw Inorder to fetch the data from Publisher db pub , table t1 to sub db in Subscriber   follow the line No  command executed on publisher side : 92

After executing the command of line no 92 on publisher side  next is to create subscription on subscriber side
sub=# CREATE SUBSCRIPTION my_sub CONNECTION 'host=localhost port=9999 dbname=pub user=repuser password=password' PUBLICATION my_pub;
NOTICE:  created replication slot "my_sub" on publisher
CREATE SUBSCRIPTION

Once subscription is created you can see auto log will start generating on publisher side and next check details in tabble t1 of sub db at subscriber side 
sub=# select *from t1;

Now any chnages made in publisher db will be auto replicated to sub db in subscriber 
follow the lines 94 - 100 to observe changes happening in subscriber side 

sub=# select *from t1;
 a | b  
---+----
 1 | 10
 2 | 20
(2 rows)

sub=# select *from t1;
 a  |  b   
----+------
  1 |   10
  2 |   20
 25 | 1000
(3 rows)

sub=# select *from t1;
 a  |  b   
----+------
  1 |   10
 25 | 1000
 29 |   20














