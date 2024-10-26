---
layout: post
title: Mysql conditional INSERTS
date: '2016-10-06 02:09:10'
tags:
- mysql
---

Every now again it's useful to have slightly more control over a MySQL insert than simply making it idempotent via the `IGNORE` keyword.  For example;
```language-sql
INSERT IGNORE INTO foo (id, column_bar) values (1, 'aaa'),  (2, 'bbb');
```
The `IGNORE` keyword will simply skip over any primary or unique key constraint violations, essentially making the above statement idempotent assuming a primary key on `id`.

However let us suppose we have a data set without a primary key or more precisely the data we want to insert has more complex conditional requirements.   Unfortunately MySQL's `INSERT` statement does not directly allow for greater selectivity but the `SELECT` statement does allowing us to take advantage of the `INSERT...SELECT...` form.

```language-sql
CREATE TEMPORARY TABLE tmp_users LIKE users;
INSERT INTO tmp_users VALUES ( ....... *default user list*)

INSERT INTO users SELECT * FROM tmp_users WHERE .... <conditional logic here>

-- Optional as temporary tables only exist for duration of session.
DROP TABLE tmp_users;  
```

Admittedly this is a contrived example where there would likely be a `UNIQUE` key on `username` which could be taken advantage of in the `INSERT IGNORE...` statement format.  However this is simply to illustrate how more complex logic can be wrapped around an `INSERT` statement when needed without any supporting code or stored procedures.

See [here](https://dev.mysql.com/doc/refman/5.7/en/insert.html) for full documentation on Mysql `INSERT` statement.