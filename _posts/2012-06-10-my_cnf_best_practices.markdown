---
layout: post
title: "~/.my.cnf Best Practices"
date: '2012-06-10 14:32:00'
tags:
- mysql_client
---

I recently gave a lunch n learn on some of the trick and tips the humble mysql client has to offer, making life a little easier at the command prompt.  I've been pondering a blog of some sort for a while and this seemed a great subject to start with and see where it goes.  Bear in mind I make no guarantee any of what's posted here will be useful or recommended for your particular setup or way of working.   These are simply things that have made my professional life a little easier.  Anyway without much ado..... 

## ~/.my.cnf best practices

Just like customizing your shell prompt can be extremely useful, so can customizing the mysql prompt.  Out of the box it's usually configured to include hostname, username and current database.  Pretty useful defaults. 
However if you're connecting to a database locally the hostname (`\h` in the prompt definition) is reported as `localhost`.  Not particularly useful.  To work around this I have started to hard code in the hostname in the mysql prompt definition.  It's klunky but if you only have a small number of servers, it's a low cost win.  At large numbers of servers you probably want to look at scripting/templating the generation of your configuration file on each server.  If you remotely connect to databases this shouldn't be an issue but security/firewall restrictions do not always permit this. 

Setting a default username and password help reduce both mistakes and typing.  It's worth noting that default usernames and passwords are specific to each mysql client.  I.e. a separate entry is needed for `mysql`, `mysqldump`, `mysqladmin` etc to set appropriate default user credentials.  Setting the default username and password for each mysql client program also streamlines & simplifies the secure scripting use of those programs. 

The mysql CLI does support tab completion for both table names and fields.  Sadly database tab complete does not seem to be supported.  Tab completion may or may not be on by default depending on build/configuration etc.  If you want to force table completion specify auto-hash in your configuration file. 

Most mysql instances have a 'go-to' database.  Again, this can be set as a default in the `my.cnf` file with the database setting. 

An example using everything mentions so far might look like this. 

```
 [mysql] 
 user=testuser 
 password=testpassword 
 prompt=\\u@myservername [\\d] > 
 auto-hash 
 database=someproductiondb 
 
 [mysqladmin] 
 user=anothertestuser 
 password=anothertestpassword
```