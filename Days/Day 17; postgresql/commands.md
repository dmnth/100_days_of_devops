We are tasked to create database, user and grant full access on created database to this user.

```sql
postgres=# \l
                             List of databases
   Name    |  Owner   | Encoding | Collate | Ctype  |   Access privileges   
-----------+----------+----------+---------+--------+-----------------------
 postgres  | postgres | UTF8     | C.utf8  | C.utf8 | 
 template0 | postgres | UTF8     | C.utf8  | C.utf8 | =c/postgres          +
           |          |          |         |        | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.utf8  | C.utf8 | =c/postgres          +
           |          |          |         |        | postgres=CTc/postgres
(3 rows)

postgres=# CREATE USER kodekloud_aim WITH PASSWORD GyQkFRVNr3;
ERROR:  syntax error at or near "GyQkFRVNr3"
LINE 1: CREATE USER kodekloud_aim WITH PASSWORD GyQkFRVNr3;
                                                ^
postgres=# CREATE USER kodekloud_aim WITH PASSWORD 'GyQkFRVNr3';
CREATE ROLE
postgres=# CREATE DATABASE kodekloud_db9 OWNER kodekloud_aim;
CREATE DATABASE
postgres=# GRANT ALL PRIVILEGES ON kodekloud_db9 TO kodekloud_aim;
ERROR:  relation "kodekloud_db9" does not exist
postgres=# GRANT ALL ON DATABASE kodekloud_db9 TO kodekloud_aim;
GRANT
postgres=# \db
       List of tablespaces
    Name    |  Owner   | Location 
------------+----------+----------
 pg_default | postgres | 
 pg_global  | postgres | 
(2 rows)

postgres=# \dv
Did not find any relations.
postgres=# \l
                                       List of databases
     Name      |     Owner     | Encoding | Collate | Ctype  |        Access privileges        
---------------+---------------+----------+---------+--------+---------------------------------
 kodekloud_db9 | kodekloud_aim | UTF8     | C.utf8  | C.utf8 | =Tc/kodekloud_aim              +
               |               |          |         |        | kodekloud_aim=CTc/kodekloud_aim
 postgres      | postgres      | UTF8     | C.utf8  | C.utf8 | 
 template0     | postgres      | UTF8     | C.utf8  | C.utf8 | =c/postgres                    +
               |               |          |         |        | postgres=CTc/postgres
 template1     | postgres      | UTF8     | C.utf8  | C.utf8 | =c/postgres                    +
               |               |          |         |        | postgres=CTc/postgres
(4 rows)
```
