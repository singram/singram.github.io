---
layout: post
title: Tips for working with Mysql at the CLI
date: '2012-06-30 13:36:00'
tags:
- mysql
- mysql_client
---

## Warnings

Warnings by default are hidden. The last warning can be displayed by the SHOW WARNING command. This is somewhat cumbersome. `\W` will turn on the automatic display of all warnings. `\w` will hide warnings by default again. If you wish this to be the default behavior in your mysql client put show-warnings in your `~/my.cnf` file 

    [mysql] 
    show-warnings

This is particularly useful if you are running in a non-strict sql mode that is very forgiving. 

## Safe mode

Safe updates is an interesting mode for working in production environments. When active this mode will prevent any UPDATE or DELETE statement from executing without either a LIMIT or WHERE clause present. This mode can be activated by the following statement 

    SET SQL_SAFE_UPDATES=1;

and similarly deactivated with 

    SET SQL_SAFE_UPDATES=0;

If you desire this to be the default mode for your mysql client simple put safe-updates in your `~/my.cnf` file  (This can also be configured at the server level with the `sql_safe_updates=1` directive) 
    
    [mysql] 
    safe-updates

## Vertical results

Wide result sets and status commands do not lend themselves well to the standard row based output format. For these instances, listing each row vertically by field may make for easier reading. This can be specified by terminating a statement with `\G` rather than `;`. For instance 

    SHOW SLAVE STATUS\G

## Paging large result sets

Following the topic of making results easier to digest you can employ your favorite paging utility directly within the mysql client allowing you to navigate and search large data set results (depending on the functionality of the utility program you have specified of course). This can be set as follows 

    mysql> pager less 
    mysql> SELECT * FROM super_large_table;

This sets the 'less' command to page through large data sets, 'more' is another obvious choice (pager more). You can disable this functionality with the 'nopager' command. You can also specify this as the default client behavior in your `~/my.cnf`. 

## System access

Occasionally when inside the mysql client I find myself needing to step back outside to the bash prompt to locate a file to source in or change directory. Exiting out of the mysql session or opening another ssh session is less than ideal. A better solution is to work with your OS within the mysql client through the use of the system command which is also aliased to `\!` 

    mysql> system ls -l 
    mysql> \! pwd

You can even spawn off a completely new bash session from within the mysql client. 

    mysql> \! bash

## External statement editing

We've all done it. Finished typing a particularly long sql statement only to find there's a typo in the middle. Unfortunately the client is not the most nimble of clients to edit in. Wouldn't it be great to take that statement and be able to edit it in your favorite text editor? By terminating the statement with `\e` you will open up your editor (specified by your `EDITOR` environment variable) with the current statement available for editing. Once you have made the corrections, save and exit the editor. Bear in mind the corrected statement WILL NOT appear in the client. At this point you will need to enter `\g` to execute the edited statement.  Not particularly intuitive but useful none the less. 

[here](http://dev.mysql.com/doc/refman/5.1/en/mysql-commands.html)