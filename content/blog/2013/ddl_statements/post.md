---
title: "DDL statements in MySQL 5.x with row-based replication"
date: "2013-06-02"
author: "Joshua Prunier"
tags: ["planet_mysql", "mysql", "ddl", "dml", "replication"]
weight: 1
---
In the replication topology I manage there are many layers of replication filters that prune data at the database and&nbsp;in a few places table level. The way MySQL replicates <a href="http://dev.mysql.com/doc/refman/5.5/en/sql-syntax-data-definition.html">Data Definition Language</a> (create, alter, drop) statements differs from how <a href="http://dev.mysql.com/doc/refman/5.5/en/sql-syntax-data-manipulation.html">Data Manipulation Language</a> (insert, update, delete) statements are handled with row-based replication. I often need to fix broken replication due to a lack of understanding of these subtle differences.

With row-based replication DML statements focus directly on the table being modified. DDL on the other hand always uses statement-based replication and is tied to what is known in MySQL as the "default database". The default database is the schema/database currently in use when a DDL statement is executed. The only information I was able to find regarding this in the MySQL documentation is the second to last paragraph <a href="http://dev.mysql.com/doc/refman/5.5/en/binary-log-setting.html">here</a>. If do-db or ignore-db filters are configured on slaves, replication can break when DDL statements are used in certain scenarios. Table level or wildcard replication filters can also add to the potential breakage but I will save that for a separate blog post.

A few examples should make things clear. The slave in the following examples is configured with "replicate-ignore-db = test". That means it will replicate all objects from its master except for anything in the "test" schema/database. The test environment is reset between examples. I have changed the prompt via the my.cnf to show the default database and statement counter: prompt = '\h(\d):\c&gt; '


<h4>Example #1</h4>

On the master we use the test schema and create a table named ddl_test.

<pre>
master((none)):1> use test;
Database changed
master(test):2> show tables;
Empty set (0.00 sec)

master(test):3> create table ddl_test(id tinyint unsigned auto_increment primary key);
Query OK, 0 rows affected (0.27 sec)

master(test):4> show tables;
+----------------+
| Tables_in_test |
+----------------+
| ddl_test       |
+----------------+
1 row in set (0.00 sec)

master(test):5>
</pre>

The ddl_test table does not replicate to the test schema which is the expected behavior.

<pre>
slave((none)):1> use test;
Database changed
slave(test):2> show tables;
Empty set (0.00 sec)

slave(test):3>
</pre>

<h4>
Example #2</h4>

This time on the master we use the world schema, create the ddl_test table in the test schema by way of a fully qualified table name and insert a row to it.

<pre><code>
master((none)):1> use world;
Database changed
master(world):2> create table test.ddl_test(id tinyint unsigned auto_increment primary key);
Query OK, 0 rows affected (0.28 sec)

master(world):3> insert into test.ddl_test(id) values(1);
Query OK, 1 row affected (0.00 sec)

master(world):4> use test;
Database changed
master(test):5> show tables;
+----------------+
| Tables_in_test |
+----------------+
| ddl_test       |
+----------------+
1 row in set (0.00 sec)

master(test):6> select * from ddl_test;
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)

master(test):7>
</code></pre>

In this case the table creation is replicated because create is a DDL statement and the world schema was the default database when the statement was issued. Even though the default database was world, an insert does not replicate to the slave because it is a DML statement and the slave is configured to ignore events on objects in the test schema.

<code><pre>
slave((none)):1> use test;
Database changed
slave(test):2> show tables;
+----------------+
| Tables_in_test |
+----------------+
| ddl_test       |
+----------------+
1 row in set (0.00 sec)

slave(test):3> select * from ddl_test;
Empty set (0.00 sec)

slave(test):4>
</pre></code>

<h4>
Example #3</h4>

Now we create the ddl_test table while test is the default database, then alter the table to add a column while world is the default database.
<code><pre>
master((none)):1> use test;
Database changed
master(test):2> create table test.ddl_test(id tinyint unsigned auto_increment primary key);
Query OK, 0 rows affected (0.30 sec)

master(test):3> use world;
Database changed
master(world):4> alter table test.ddl_test add column newcol varchar(1);
Query OK, 0 rows affected (0.48 sec)
Records: 0  Duplicates: 0  Warnings: 0

master(world):5> show create table test.ddl_test\G
*************************** 1. row ***************************
       Table: ddl_test
Create Table: CREATE TABLE `ddl_test` (
  `id` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
  `newcol` varchar(1) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.06 sec)

master(world):6>
</pre></code>

Ouch! The result is broken replication on the slave. The reason is the original table create did not replicate but the additional alter statement does.
<code><pre>
slave((none)):1> use test;
Database changed
slave(test):2> show tables;
Empty set (0.00 sec)

slave(test):3> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.101
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000002
          Read_Master_Log_Pos: 1132
               Relay_Log_File: slave-relay-bin.000010
                Relay_Log_Pos: 1164
        Relay_Master_Log_File: master-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: test
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1146
                   Last_Error: Error 'Table 'test.ddl_test' doesn't exist' on query. Default database: 'world'. Query: 'alter table test.ddl_test add column newcol varchar(1)'
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1014
              Relay_Log_Space: 1589
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1146
               Last_SQL_Error: Error 'Table 'test.ddl_test' doesn't exist' on query. Default database: 'world'. Query: 'alter table test.ddl_test add column newcol varchar(1)'
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
1 row in set (0.00 sec)

slave(test):4>
</pre></code>

The error message shows exactly what went wrong. Table does not exist is pretty clear. The default database is world, the query references the test.ddl_test table and shows the alter table command that was attempted. Show slave status output also shows us the Replicate_Ignore_DB configuration has been set on the slave to ignore all DDL and DML statements against the test schema.

The next course of action is to get replication going again. We have deduced the alter table statement should not have replicated so we simply skip the event using the sql_slave_skip_counter variable. 
<code><pre>
slave(test):4> set global sql_slave_skip_counter=1;
Query OK, 0 rows affected (0.00 sec)

slave(test):5> start slave;
Query OK, 0 rows affected (0.00 sec)

slave(test):6> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.101
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000002
          Read_Master_Log_Pos: 1132
               Relay_Log_File: slave-relay-bin.000010
                Relay_Log_Pos: 1282
        Relay_Master_Log_File: master-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: test
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1132
              Relay_Log_Space: 1589
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
1 row in set (0.00 sec)

slave(test):7>
</pre></code>

Not directly related to the theme of this post but worth mentioning is the unique schema in MySQL known as none or null. This schema can not be used directly since it is the absence of having a default database. The only way I know of acquiring null schema is to connect to mysql without the -D option or by dropping the schema that is currently in use. How DDL and DML statements are handled remains the same with or without a default database.

Hopefully the examples above make the rules that govern replication of DDL and DML statements clearer. The potential impact on a dba really depends on the size and configuration of their replication topology. The more databases and filters in place to limit and/or direct what is replicated, the greater the chance replication can break. Number of users manipulating database objects, their knowledge of MySQL and the replication topology they are working within is also a big factor.
