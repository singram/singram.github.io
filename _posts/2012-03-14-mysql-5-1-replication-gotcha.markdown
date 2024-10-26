---
layout: post
title: Mysql 5.1 table replication gotcha
date: '2012-03-14 00:07:00'
tags:
- mysql
- replication
---

So I came across an interesting scenario lately regarding mysql 5.1 replication (statement based).

One would think if the Innodb engine fails to initialize upon startup for some reason, that replication would fail with an error when creating a table with the Innodb engine specified explicitly via a statement received from the master. 
Maybe you changed the log file size in your config without removing the existing innodb log files so they could regenerate and neglected to check that the engine was up and running upon a subsequent reboot.

If you were expecting a error on the mysql slave thread you're expectations would be wrong.  In this scenario all tables with ENGINE=Innodb in the statement actually fall back to the default which is usually MyIsam on 5.1 and earlier releases.  This obviously causes some unexpected problems in statement based replication and when trying to resolve data drift since the engines behave radically different.

Solutions to avoid config and structural drift within a replicating cluster are 

- Periodically check the mysqld.err.log file across all servers.
- Use a tool to compare configurations across servers
  pt-mysql-summary is excellent but you could simply compare the results of running mysqladmin variables.
- Use a tool to verify the structure of your database (tables, indexes, column definitions, triggers, etc).