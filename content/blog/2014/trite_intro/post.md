---
title: "Introducing Trite: A tool for automating import of InnoDB tablespaces"
date: "2014-02-14"
author: "Joshua Prunier"
tags: ["planet_mysql", "mysql", "trite", "go", "percona", "xtrabackup", "backup", "innodb"]
weight: 1
---

{{% imgleft src="/static/img/gopher_dolphin.jpg" %}}

<p>
Mysqldump is a fantastic tool for backing up and restoring small and medium sized MySQL tables and databases quickly. However, when databases surge into the multi-terabyte range restoring from logical backups is inefficient. It can take a significant amount of time to insert a hundred million plus rows to a single table, even with very fast I/O. Programs like MySQL Enterprise Backup and Percona XtraBackup allow non-blocking binary copies of your InnoDB tables to be taken while it is online and processing requests. XtraBackup also has an export feature that allows InnoDB file per tablespaces to be detached from the shared table space and imported to a completely different MySQL instance.
</p>

<p>
The necessary steps to export and import InnoDB tables are in the XtraBackup documentation <a href="http://www.percona.com/doc/percona-xtrabackup/2.1/innobackupex/importing_exporting_tables_ibk.html" target="_blank">here</a>. Although fairly simple and straight forward it requires a bit too much typing if you want to move more than a couple of tables. To automate the process I created a new open source project named <a href="https://github.com/joshuaprunier/trite" target="_blank">Trite</a>.
</p>

<p>
Trite stands for <b>TR</b>ansport <b>I</b>nnodb <b>T</b>ables <b>E</b>fficiently and is also a nod to the repetitive manual steps required to move tablespaces. Trite is a client/server written in 100% <a href="http://golang.org/" target="_blank">Go</a> that leverages XtraBackups's import/export feature and aids a DBA in creating, recovering or refreshing databases. The initial problem that led me to create Trite is a common development database environment where I work. Multiple applications share the same database servers which requires each applications data be refreshed at different intervals. Before Trite we used mysqldump and it took the majority of a weekend for the data refresh to complete, even with good hardware and my.cnf's tuned for performance. With Trite we get the same results in a fraction of the time!
</p>

<p>
Trite has three modes of operation: dump, server and client. Dump mode writes out create statements for schemas, tables, stored procedures, functions, triggers and views. Server mode makes the XtraBackup and dump create files accessible via a Go powered HTTP server. Client mode downloads the create statements and XtraBackup files from a Trite server to recreate database objects.
</p>

<br>
<p>
Twelve gigabytes of <a href="http://imdbpy.sourceforge.net/" target="_blank">IMDB</a> data will make a suitably sized database for an example. The hardware used is a woefully under powered laptop; 4 gigs of RAM, 32bit CPU, slow SATA hard drive. Before each data load the imdb schema was removed and MySQL restarted to wipe caches. It took just over two hours to restore from the dump file. This is with innodb_doublewrite turned off to try and speed up all the inserts.
</p>

<pre>
jprunier@beefeater:~/Downloads$ ls -alh imdb.sql
-rw-rw-r-- 1 jprunier jprunier 4.8G Jan 24 18:41 imdb.sql

jprunier@beefeater:~/Downloads$ time mysql -uroot -p < imdb.sql
Enter password: 

real	130m55.167s
user	2m16.052s
sys		0m9.020s

jprunier@beefeater:~/Downloads$ sudo du -sh /var/lib/mysql/imdb
[sudo] password for jprunier: 
12G	/var/lib/mysql/imdb
</pre>

<br>
<p>
A binary backup of the database took 8 minutes with XtraBackup.
</p>

<pre>
root@beefeater:~# time innobackupex --user=root --password= /root

... output omitted ...

innobackupex: Backup created in directory '/root/2014-01-25_14-22-06'
140125 14:30:29  innobackupex: Connection to database server closed
140125 14:30:29  innobackupex: completed OK!

real	8m23.118s
user	0m42.076s
sys		0m31.524s
</pre>

<br>
<p>
And 16 seconds to prepare the tables for export. This will take longer depending on the amount of changes the database receives during the backup. The apply can be sped up by giving the innobackupex process more memory.
</p>

<pre>
root@beefeater:~# time innobackupex --apply-log --use-memory=1G --export /root/2014-01-25_14-22-06

... output omitted ...

140125 14:33:03  InnoDB: Starting shutdown...
140125 14:33:07  InnoDB: Shutdown completed; log sequence number 70808870412
140125 14:33:07  innobackupex: completed OK!

real	0m16.310s
user	0m0.900s
sys		0m0.620s
</pre>

<br>
<p>
A dump of the database objects should be done as close to the backup as possible. If the Trite create statement from a dump differs significantly from the copy of table taken with XtraBackup the table may fail to import.
</p>

<pre>
root@beefeater:~# ./trite -dump -user=root -password= -host=localhost
Dumping to: /root/localhost_dump20140125173337

Enter password: 

imdb: 21 tables, 0 procedures, 0 functions, 0 triggers, 0 views

22 total objects dumped

Total runtime = 2.19331264s
</pre>

<br>
<p>
In this example the Trite server is being run on the same machine as the database being restored. If your database is very large the backup, create dumps and Trite server will probably be on a different host.
</p>

<pre>
root@beefeater:~# screen -S trite_server
root@beefeater:~# ./trite -server -dump_path=/root/localhost_dump20140125173337 -backup_path=/root/2014-01-25_14-22-06

Starting server listening on port 12000
</pre>

<br>
<p>
The Trite client was able to restore the imdb database in 13 minutes. Much faster than doing all those inserts over with mysqldump. You can run the client with multiple worker threads, the default is single threaded. The optimal number of workers depends on the size of your tables, how fast your disks are and how fast your network is if the server is remote. In the case of the laptop, multiple workers actually makes the restore time slower.
</p>

<pre>
root@beefeater:~# ./trite -client -user=root -password= -socket=/var/lib/mysql/mysql.sock -server_host=localhost
Enter password: 
imdb.link_type has been restored
imdb.comp_cast_type has been restored
imdb.movie_info has been restored
imdb.aka_name has been restored
imdb.complete_cast has been restored
imdb.movie_link has been restored
imdb.company_name has been restored
imdb.char_name has been restored
imdb.name has been restored
imdb.person_info has been restored
imdb.keyword has been restored
imdb.cast_info has been restored
imdb.role_type has been restored
imdb.aka_title has been restored
imdb.kind_type has been restored
imdb.movie_info_idx has been restored
imdb.movie_keyword has been restored
imdb.company_type has been restored
imdb.info_type has been restored
imdb.title has been restored
imdb.movie_companies has been restored
Applying triggers for imdb
Applying views for imdb
Applying procedures for imdb
Applying functions for imdb

Total runtime = 13m3.881954844s
</pre>

<br>
<p>
Xtrabackup can restore the entire database just as quick but you need to restore the entire backup. The power of Trite is being able to customize a restore. By removing files from the dump directory you can restore just spcific schemas or tables. You can also restore to a database that contains tables while it is online and responding to queries. Some of the features I am planning on adding next are compression for transfer across slow networks and replication support so only objects conforming to a slaves replication filters will be applied.
</p>

<br>
<p>
Trite does have some limitations. The full list can be found on the <a href="http://github.com/joshuaprunier/trite" target="_blank">GitHub</a> page but the top ones are:
</p>

<div>
  <ul>
    <li>InnoDB file per table is required.</li>
    <li>The database you are restoring to must be running Percona server 5.1, 5.5, 5.6 or Oracle MySQL 5.6 or MariaDB 5.5, 10.</li>
    <li>Windows is not currently supported due to a few Linux specific lines of code.</li>
  </ul>
</div>

<p>
If you think Trite might benefit you and your organization give it a try and send me any feedback you might have. I have tried hard to make Trite efficient and bug free but there might be issues when used with certain configurations. Be sure to test Trite on non-critical databases before use in production. If you encounter any bugs please open an issue via GitHub.
</p>
