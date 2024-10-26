---
layout: post
title: Client side logging in mysql
date: '2012-06-18 18:21:00'
tags:
- mysql
- mysql_client
- logging
---

When using the mysql CLI users typically have two options when it comes to logging at the client side. 

1. Statement only logging can be found in `~/.mysql_history` (see [here](http://dev.mysql.com/doc/refman/5.1/en/mysql-history-file.html)
2. Statement & result, etc logging can be obtained through the use of the `tee` and `notee` commands. The `tee` command specifies a file to record all subsequent output to, including statement and results. The `notee` command terminates the recording of the session.

```
mysql> tee /home/someuser/session_log_123.txt 
mysql> SELECT * FROM .....; 
mysql> ....some results 
mysql> notee 

mysql> \! cat /home/someuser/session_log_123.txt
```