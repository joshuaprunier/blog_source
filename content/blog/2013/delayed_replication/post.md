---
title: "Delayed row-based replication with large tables lacking a primary key"
date: "2013-05-13"
author: "Joshua Prunier"
tags: ["planet_mysql", "mysql", "innodb", "primary key", "rbr", "replication"]
weight: 1
---

I configure all our master databases to use row-based binary logging where I work. In my opinion it is a much safer option than statement-based replication. The advantages and disadvantages of both types of MySQL replication are detailed in the online documentation&nbsp;<a href="https://dev.mysql.com/doc/refman/5.5/en/replication-sbr-rbr.html" rel="nofollow" target="_blank">here</a>. You can't view the events a slave is applying directly with <i><b>'show processlist'</b></i> but by issuing <i><b>'show open tables where in use'</b></i> you can detect what table is receiving the attention of the SQL thread. If you need more information the mysqlbinlog command must be used to decode the slaves relay logs or masters binary logs.

Our developers often change a lot of rows with a single update statement. This usually results in some reasonable replication lag on downstream slaves.&nbsp;Occasionally&nbsp;the lag continues to grow and eventually nagios complains. Investigating the lag I sometimes discover the root of the problem is due to millions of rows updated on a table with no primary key. Putting a primary key constraint on a table is just good practice, especially on InnoDB tables. This is really necessary for any large table that will be replicated.

To show what replication is actually doing I have cloned the sakila.film table, omitting the indexes, inserted all its rows and finally updated the description column for all 1,000 rows in the new table.

<pre><code>
beefeater(test)> show master status;
+----------------------+----------+--------------+------------------+
| File                 | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+----------------------+----------+--------------+------------------+
| beefeater-bin.000001 |      107 |              |                  |
+----------------------+----------+--------------+------------------+
1 row in set (0.00 sec)

beefeater(test)> use test;
Database changed
beefeater(test)> CREATE TABLE test.film (
    ->   film_id smallint(5) unsigned NOT NULL,
    ->   title varchar(255) NOT NULL,
    ->   description text,
    ->   release_year year(4) DEFAULT NULL,
    ->   language_id tinyint(3) unsigned NOT NULL,
    ->   original_language_id tinyint(3) unsigned DEFAULT NULL,
    ->   rental_duration tinyint(3) unsigned NOT NULL DEFAULT '3',
    ->   rental_rate decimal(4,2) NOT NULL DEFAULT '4.99',
    ->   length smallint(5) unsigned DEFAULT NULL,
    ->   replacement_cost decimal(5,2) NOT NULL DEFAULT '19.99',
    ->   rating enum('G','PG','PG-13','R','NC-17') DEFAULT 'G',
    ->   special_features set('Trailers','Commentaries','Deleted Scenes','Behind the Scenes') DEFAULT NULL,
    ->   last_update timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.23 sec)

beefeater(test)> insert into test.film select * from sakila.film;
Query OK, 1000 rows affected (0.37 sec)
Records: 1000  Duplicates: 0  Warnings: 0

beefeater(test)> show master status;
+----------------------+----------+--------------+------------------+
| File                 | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+----------------------+----------+--------------+------------------+
| beefeater-bin.000001 |   137389 |              |                  |
+----------------------+----------+--------------+------------------+
1 row in set (0.00 sec)

beefeater(test)> update test.film set description = '';
Query OK, 1000 rows affected (0.05 sec)
Rows matched: 1000  Changed: 1000  Warnings: 0

beefeater(test)> show master status;
+----------------------+----------+--------------+------------------+
| File                 | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+----------------------+----------+--------------+------------------+
| beefeater-bin.000001 |   313847 |              |                  |
+----------------------+----------+--------------+------------------+
1 row in set (0.00 sec)

</code></pre>

I have edited the output of mysqlbinlog to only show the entries related to the first row created by the insert and update&nbsp;statements&nbsp;above. The&nbsp;@ symbol followed by a number is a mapping to the columns in the table. So&nbsp;@1=film_id,&nbsp;@2=title, @3=description and so on. Note that the update statement records a before and after picture of the row. This is can be used in a pinch to fix mistaken updates if the damage is small instead of having to restore from backups.

<pre><code>
mysqlbinlog --base64-output=auto beefeater-bin.000001 | more
...
### INSERT INTO test.film
### SET
###   @1=1
###   @2='ACADEMY DINOSAUR'
###   @3='A Epic Drama of a Feminist And a Mad Scientist who must Battle a Teacher in The Canadian R
ockies'
###   @4=2006
###   @5=1
###   @6=NULL
###   @7=6
###   @8=990000000
###   @9=86
###   @10=000000020.990000000
###   @11=2
###   @12=b'00001100'
###   @13=1139997822
...

root@beefeater:/var/lib/mysql# mysqlbinlog --base64-output=auto --start-position=137389 beefeater-bin.000001 | more
...
### UPDATE test.film
### WHERE
###   @1=1
###   @2='ACADEMY DINOSAUR'
###   @3='A Epic Drama of a Feminist And a Mad Scientist who must Battle a Teacher in The Canadian R
ockies'
###   @4=2006
###   @5=1
###   @6=NULL
###   @7=6
###   @8=990000000
###   @9=86
###   @10=000000020.990000000
###   @11=2
###   @12=b'00001100'
###   @13=1139997822
### SET
###   @1=1
###   @2='ACADEMY DINOSAUR'
###   @3=''
###   @4=2006
###   @5=1
###   @6=NULL
###   @7=6
###   @8=990000000
###   @9=86
###   @10=000000020.990000000
###   @11=2
###   @12=b'00001100'
###   @13=1368396106
...
</code></pre>

So row-based replication is performing as named and creating a binary log event for each row affected. My single insert and update statement on the master then became 1,000 separate events on the slave.

Digging in to the MySQL source code I was unable to confirm exactly how the SQL thread applies relay log events on a slave. I assume it is similar to what happens when a normal update statement is executed on a table with no indexes. The server must perform a full table scan to locate a single row. For a table with a million plus rows a million full table scans is expensive! A primary key or suitable unique index will prevent this type of problem.
