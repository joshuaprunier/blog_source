---
title: "Trite Bug Fixes and Updates"
date: "2016-04-11"
author: "Joshua Prunier"
tags: ["mysql", "trite", "go", "percona", "xtrabackup", "backup", "inboard"]
weight: 1
---

{{% imgleft src="/static/img/gopher_dolphin.jpg" %}}

<p>
I originally released <a href="https://github.com/joshuaprunier/trite" target="_blank">Trite</a> a little over two years ago. Since then I have continued to fix bugs and add new functionality as spare time allowed. The primary reason for this is that Trite is a tool I often use where I work but another important reason is that I am constantly working to improve my knowledge and abilities. 
</p>
<p>
The following items are some of the major changes and new functionality available:
</p>

<style> ul { list-style-position:inside; } </style>
<div>
  <ul>
	  <li>Code is more idiomatic Go</li>
	  <li>Linux, OSX & Windows support</li>
	  <li>Server root landing page</li>
	  <li>Table download and import logic modified</li>
	  <li>Reformatted progress display</li>
	  <li>MySQL Unicode object translation</li>
	  <li>Improved error handling and logging</li>
	  <li>Support for TLS & clear text configured databases</li>
	  <li>Optional compression for slow networks</li>
  </ul>
</div>

<br><p><b>Code is more idiomatic Go</b></p>
<p>
This isn't a bug fix or new functionality but is something I consider important. Trite was my first major project in Go so naturally I did some things incorrect or weird being a novice with the language. Making the Trite code more idiomatic Go was about going back and refactoring to make it more readable and hopefully more extensible.
</p>

<br><p><b>Linux, OSX & Windows support</b></p>
<p>
One of Go's great features is cross platform compilation. However, some things, like dealing with standard in/out/err is a little different between Windows and Linux/OSX. By switching to the <a href="https://godoc.org/golang.org/x/crypto/ssh/terminal" target="_blank">ssh/terminal</a> sub-repo Trite is better able to handle prompting the user for a password in a secure manner and also reattaching stdout if the user decides to ctrl+c during the prompting phase. Also windows does not have the same concept of user and group ownership for files that Linux/OSX does and so there are some OS specific exceptions. The majority of the code is the same across platforms though.
</p>

<br><p><b>Server root landing page</b></p>
<p>The Trite server now has a root landing page with links to the structure dump .sql and XtraBackup files. Before you had to travel directly to the /tables or /backups path. Not a huge deal but a nice convenience.</p>

<br><p><b>Table download and import logic modified</b></p>
<p>The way I originally implemented Trite's download and apply logic never really sat well with me. It worked but it was messy, especially if exceptions were encountered during run time. I didn't really understand how to use Go's channels and deal with concurrency properly at the time. With the prior logic, you specified a number of worker threads and Trite would begin downloading that many files at once. Then due to file size differences the downloads and imports would begin to offset. It was a crude attempt to fix a single threaded performance problem where for a single worker during the table import phase nothing was being downloaded. Once I understood this problem properly and had more experience with Go's concurrency primitives a better solution became clear. Create a download queue that is single threaded and allow table imports to perform concurrently if necessary. So the NIC is always in use and not over saturated with multiple file downloads. An additional benefit to this new logic was the ability to clean up and handle errors better when they occur.</p>

<br><p><b>Reformatted progress display</b></p>
<p>The new progress display was a bit tricky to write but makes for a much better user experience. The display is closely related to the new table download and import logic. The problem with the previous logic was that downloading and importing was happening concurrently for many tables. It only made sense to notify when a table had been restored. Otherwise the display output would have been a jumbled mess. The progress display is now single threaded. A table will begin with a status of "Downloading," then "Applying" and finally "Restored." A tables status only occupies a single line and that information is rewritten when the status is updated. If a problem occurs then a status of "ERROR" is shown and information is written to the error file. So if a very large table takes a while to download and apply, the display will stay on an applying status for that table, even though smaller tables may be downloading and importing in the background. As soon as the long running table has finished importing the display will quickly catch up, printing any other tables that may have finished already, stopping on the table currently being processed and its status.

The display will also now show the download progress for large files. The Trite client defaults to showing the downloaded percentage for any file over 5 gigabytes. The size a file must be for download progress to be shown can be adjusted with the <i>-progressLimit</i> flag. Some very large tables can take dozens of minutes or hours to download even on a fast internal network and it's nice to see that Trite is in fact still running properly.
</p>

<br><p><b>MySQL Unicode object translation</b></p>
MySQL encodes certain Unicode characters on the file system. If you create a schema or table with the omega character "Ω" for example, MySQL will create a directory or file named "@7P" on disk even though the character is properly visible within the database. I referenced the MySQL source code to determine how MySQL does this and created a <a href="https://github.com/joshuaprunier/mysqlUTF8" target="_blank">mysqlUTF8</a> library in Go for Trite to import. Trite can now properly work with Unicode MySQL objects and find the correct path and file names within an XtraBackup or MySQL data directory. This wasn't something I personally had to deal with where I work but knew it could affect some users and wanted to make sure it was not an issue for non-native English speakers that might want to use Trite.
<p>
</p>

<br><p><b>Improved error handling and logging</b></p>
<p>
If something went wrong Trite used to just dump a stack trace to the user. This was not very helpful at all. Trite now does its best to detect when an error occurs, display that an error occurred, log the error and some database information to a file and try to continue processing. That file by default is "trite.err" in the directory where trite is invoked. The <i>-errorLog</i> flag allows you to explicitly set the location and name of the error file if desired. If a problem happens during download the partial file is removed. If a problem happens during table import or stored routine, trigger or view creation the current process list and show engine innodb status is captured and also logged to the error file to aid in debugging. 
</p>

<br><p><b>Support for TLS & clear text configured databases</b></p>
<p>
I added MySQL's clear text client side plugin functionality to the wonderful <a href="https://github.com/go-sql-driver/mysql#allowcleartextpasswords" target="_blank">Go MySQL driver</a> which Trite uses. I have configured users MySQL accounts where I work to be authenticated via LDAP and not natively within MySQL. So Trite naturally has support via the <i>-tls</i> flag to use the clear text client side plugin and it also assumes MySQL is configured with TLS to prevent passwords being compromised by tcp capture.
</p>

<br><p><b>Optional compression for slow networks</b></p>
<p>
One of Go's primary advantages is being able to take advantage of all cpu cores on a server. <a href="https://github.com/klauspost/pgzip" target="_blank">PGZIP</a> is a drop in replacement for the standard Go gzip library that allows all cpu cores to be used during compression/decompression. Using Trite to restore a 12 gigabyte database consisting of mostly text across the internet took me about 2 1/2 hours. Using on the fly compression with the new <i>-gz</i> client flag the restore time is reduced to just 30 minutes! A massive time savings and the Trite server in this case was on a fairly under powered Linux vm and the client was a macbook pro laptop. Compressed transfers could be even faster on serious enterprise grade hardware.
</p>

<br><p>
The next big thing for Trite is probably the ability to serve tables from an active MySQL database instead of an XtraBackup prepared with the --export option. MySQL 5.6/MariaDB 10 and above support this and it would be a quicker restore option than requiring an XtraBackup. This is a fair amount of work though and I'm not sure of the exact time line of when this might be accomplished.
</p>
