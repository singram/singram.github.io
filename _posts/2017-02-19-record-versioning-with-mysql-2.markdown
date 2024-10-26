---
layout: post
title: Record versioning with Mysql
date: '2017-02-19 15:57:30'
tags:
- mysql
---

The topic of how to handle record versioning came up recently in a number of projects.  This is a topic known commonly as **slowly changing dimensions**.  There are a number of approaches depending on your requirements; a good overview can be found [here](https://en.wikipedia.org/wiki/Slowly_changing_dimension).


Considerations in design

* need to persist accurate history
* number of changing fields within a record
* maintainability
* placement of business logic and number of actors on the data store

This article aims to explore a number of approaches and how [Mysql](https://www.mysql.com/)'s capabilities can be used to enforce integrity or automate the strategies where possible.  Generally I'm not a fan of stored procedures or complex triggers as this essentially divests business logic from the core code base/ service layer.  However this is a space that I wanted to explore how far Mysql triggers could assist.

Regardless of which strategy is the best fit for the requirements at hand, all scenarios should have appropriate integration tests to ensure expected behavior over time, especially as there are multiple ways to achieve these strategies.  Integration tests should cover basic scenarios such as

* Inserting new records into the system
* Updating existing records
 * Are updates reflected correctly?
 * Is history of the old record maintained?

For discussion we'll use Mysql 5.7 and model the common scenario of a supply table with 4 simple properties;

* Internal primary surrogate key
* Natural key
* Description
* Cost

Where the assumption is that the natural key is globally unique.  e.g.
```language-sql
CREATE TABLE supplies (
     id INT NOT NULL AUTO_INCREMENT,
     supply_key CHAR(10) NOT NULL,
     description CHAR(30) NOT NULL,
     cost INT DEFAULT 0,
     PRIMARY KEY (id),
     UNIQUE KEY (supply_key)
);
```

You can get started very simply if you have [docker](https://www.docker.com/) installed with the following;

```
docker run --name mysql_container --env MYSQL_ALLOW_EMPTY_PASSWORD=YES -p 3306:3306 mysql:5.7
```
This will download and run mysql in an isolated container and the following will allow you to connect
```
docker exec -it mysql_container mysql -uroot 
```
This allows you to run and utilize Mysql in a completely isolated form without polluting your host system.  It's also extremely simple to experiment between versions by simply pulling different images.  See [here](https://hub.docker.com/_/mysql/) for a complete list of offical Mysql docker images.
### Type 1 - Overwrite

Essentially, using this strategy, there is only one record per `supply_key` and fields are updated in place with no historic values retained.
Pros

* Simple to implement.  Use `INSERT IGNORE ...` or `REPLACE ....` statements depending on your needs.  This will overwrite any existing data with new data.

Cons

* Historic trends cannot be extracted from the supplies table.
* Referential integrity to the supplies table must be thought through as data can change at any point.
* Data must be copied from the supplies table to an instance of an order record to preserve state at a given point in time 

### Type 2 - Add new row

This approach involves creating a new row for a record that has changed and delineating it from existing versions.  The [Wikipeadia](https://en.wikipedia.org/wiki/Slowly_changing_dimension) article  mentions two common approaches; an incrementing `version` column grouped on `supply_key` or a combination of `start` and `end` dates.  
I favor the latter approach for 2 reasons. First, it provides temporal relevance which is useful for a wide variety of reporting and auditing reasons and second it naturally provides an easy way for determining the current record.  It's a far easier query to find out which record has a `NULL` `end` date than which `version` is the largest in a group.  For example, to get a list of current supplies with dates could be as simple as;
```language-sql
SELECT * FROM supplies WHERE ended_at IS NULL;
```
With incrementing version columns the same result could be achieved with the following more complexe statement;
```language-sql
SELECT * 
FROM supplies 
INNER JOIN 
  (SELECT supply_key, MAX(version) AS version 
   FROM supplies 
   GROUP BY supply_key) AS current
WHERE supplies.supply_key = current.supply_key 
  AND supplies.version = current.version
```
That being said, let's see how we can automate this and reduce the cognitive burden off the developer.  First let's start with a few assumptions to make this simple.

* Updates for a particular `supply_key` will be inserted in order
* Only the `start` date is supplied and is assumed to be the `end` date of the preceding record version.
* There is always a current record for a given `supply_key` with no end date.  Yes, this isn't realistic as you can never remove supplies with this constraint but we're going to roll with it for the sake of discussion.

Given these requirements our new supplies table may look something like;

```sql
DROP TABLE supplies;
CREATE TABLE supplies (
     id INT NOT NULL AUTO_INCREMENT,
     supply_key VARCHAR(10) NOT NULL,
     description VARCHAR(30) NOT NULL,
     cost INT DEFAULT 0,
     started_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
     ended_at DATETIME,
     PRIMARY KEY (id),
     UNIQUE KEY (supply_key, started_at),
     KEY (supply_key, ended_at)
);
```
Of note;

* The start date is mandatory, the end date is not since for a given `supply_key` there is always one record with a NULL `ended_at` date.
* Our `UNIQUE` key constraint now covers both the natural key for the supply and the start date.

Our automation should handle the following forms;

* A record [UPSERT](https://en.wikipedia.org/wiki/Merge_(SQL)) with an assumed start date equal to when the record was inserted
* A record UPSERT with an explicit start date
* A record UPSERT with an explicit end date
```language-sql
INSERT INTO supplies SET supply_key='A', description='foo', cost='1';
INSERT INTO supplies SET supply_key='A', description='foo', cost='1', started_at=NOW();
INSERT INTO supplies SET supply_key='A', description='foo', cost='1', started_at=DATE_SUB(NOW(), INTERVAL 1 DAY), ended_at=NOW();
```

Ideally we'd use something like the following trigger to handle this automatically.

```language-sql
DROP TRIGGER supplies_before_insert;
DROP TRIGGER supplies_after_insert;
delimiter |

CREATE TRIGGER supplies_before_insert BEFORE INSERT ON supplies
  FOR EACH ROW
  BEGIN
    SET NEW.ended_at=NULL;
  END;
|
CREATE TRIGGER supplies_after_insert AFTER INSERT ON supplies
  FOR EACH ROW
  BEGIN
    UPDATE supplies SET ended_at=NEW.started_at WHERE supply_key=NEW.supply_key AND ended_at IS NULL AND id!=NEW.id;
  END;
|

delimiter ;
```
**Unfortunately mysql does not permit updates in triggers on the same table that you inserted to.**  There reasons for this are deadlocks and infinite loops.  The update in the trigger will indeed cause the trigger to trip again and so on and so forth.

Our likely approach here is to push this on the application using something like the following;
```langauge-sql
START TRANSACTION; 
  INSERT INTO supplies 
          SET supply_key='B', description='bar', cost=2; 
  
  SELECT @end_date:=MAX(started_at) 
    FROM supplies 
   WHERE supply_key='B';
  
  UPDATE supplies 
     SET ended_at=@end_date 
   WHERE supply_key='B' 
     AND ended_at IS NULL 
     AND id!=last_insert_id(); 
COMMIT;
```
Pro's

* Orders can safely reference the supplies table and remain accurate over time.
* Simple reporting and trending
* Efficient use of covering index to locate current records.

Cons

* Your supplies table may become very large with historic data depending on the rate of change.  
* The business logic to maintain historical records must be observed by the client applications of the database meaning decentralized logic if not fronted by a service API.
* The way this particular insert query is written coupled with the automated `started_at` date field could lead to unnecessary duplication.  Consider a field that may toggle between multiple states over time.
* The above INSERT, SELECT, UPDATE combination isn't the most efficient but robustly handles automatic `started_at` values as well as specified ones.

### Type 3 - Add new attribute

In this approach the system only keeps track of the original & current values of selected fields and retains one record per supply.  In the following example the fields `description` and `cost` are of particular interest.

```language-sql
DROP TABLE supplies;
CREATE TABLE supplies (
     id INT NOT NULL AUTO_INCREMENT,
     supply_key CHAR(10) NOT NULL,
     description CHAR(30) NOT NULL,
     cost INT DEFAULT 0,
     original_description CHAR(30) DEFAULT '',
     original_cost INT DEFAULT 0,
     PRIMARY KEY (id),
     UNIQUE KEY (supply_key)
);
```

We generally want two guarantees from a system with this approach;
1. Inserts automatically fill the `original_*` fields.
2. Updates preserve the `original_*` fields.

This can be done with triggers in Mysql with the following.

```language-sql
DROP TRIGGER supplies_insert;
DROP TRIGGER supplies_update;
delimiter |

CREATE TRIGGER supplies_insert BEFORE INSERT ON supplies
  FOR EACH ROW
  BEGIN
    SET NEW.original_description = NEW.description;
    SET NEW.original_cost = NEW.cost;
  END;
|

CREATE TRIGGER supplies_update BEFORE UPDATE ON supplies
  FOR EACH ROW
  BEGIN
    SET NEW.original_description = OLD.original_description;
    SET NEW.original_cost = OLD.original_cost;
  END;
|

delimiter ;
```

With these triggers in place you can safely use standard `INSERT` and `UPDATE` statements or use the following `UPSERT` form negating the need to know upfront whether your application already contains a record for a particular `supply_key`.

```language-sql
INSERT INTO supplies SET 
    supply_key='B', description='bar', cost=2 
  ON DUPLICATE KEY UPDATE 
    cost=VALUES(cost), description=VALUES(description);
```
Pro's

* Simple to implement in either Mysql or code space
* Guarantees at the database level can be made to preserve the `original_*` fields regardless of the client interacting with the database.

Cons

* Limited use in terms of accurate record keeping.  A de-normalized copy of the supplies data at the point of use must be made for accurate record keeping.
* Trend reporting is impossible.  Only start and current costs are tracked.  Trending on de-normalized orders is not recommended since there is no guarantee that an order was made when a supply was a particular cost.

### Type 4 - Add history table

Aside from strategy [Type 2](#type2addnewrow), which retains history in the same table, the other common approach to this is to seperate historic records from current records in seperate tables;

The following creates 2 tables, a `supplies` table and a `supplies_archive` table based on the structure of the current supplies table.  The current `supplies` table still needs to know when the current record became relevant and so we need the `started_at` date.  In the `supplies_archive` we also need an `ended_at` date.

```language-sql
DROP TABLE IF EXISTS supplies;
DROP TABLE IF EXISTS supplies_archive;
CREATE TABLE supplies (
     id INT NOT NULL AUTO_INCREMENT,
     supply_key CHAR(10) NOT NULL,
     description CHAR(30) NOT NULL,
     cost INT DEFAULT 0,
     started_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
                                ON UPDATE CURRENT_TIMESTAMP,
     PRIMARY KEY (id),
     UNIQUE KEY (supply_key, started_at)
);
CREATE TABLE supplies_archive LIKE supplies;
ALTER TABLE supplies_archive
  ADD COLUMN ended_at DATETIME NOT NULL AFTER started_at;
```

Here we can set up a trigger to automatically create a new record in the `supplies_archive` table.

```language-sql
DROP TRIGGER supplies_after_update;
delimiter |

CREATE TRIGGER supplies_after_update AFTER UPDATE ON supplies
  FOR EACH ROW
  BEGIN
    IF NEW.cost != OLD.cost 
       OR NEW.description != OLD.description THEN
      INSERT INTO supplies_archive 
      SELECT NULL, NEW.supply_key, OLD.description, OLD.cost, OLD.started_at, NEW.started_at ;
    END IF;
  END;
|

delimiter ;
```

Note that this does not protect against `UPDATES` that explicitly set the `started_at` data and nothing else, which breaks our desired behavior. 

As of MySQL 5.5, you can use the SIGNAL syntax to throw an exception to assist in refining TRIGGER behavior:

```language-sql
SIGNAL sqlstate '45000' SET message_text = 'My Error Message';
```
State 45000 is a generic state representing "unhandled user-defined exception".

In any approach to a complex problem with many entry points there are workarounds.  The above solution is far from robust but is limited by the capabilities of Mysql triggers. The goal here is to provide as much consistency as possible at the database level which remains the lowest common denominator between modes of interaction, whether they are multiple clients or developers with direct access to the database.  Of course this statement is highly dependent on design and deployment environment.  For instance if you have a storage API in front of the database and ban any other method of interaction your design evaluation changes significantly.

If you have a dedicated storage API I would recommend taking the archive logic and encoding it simply in code space, forgoing all of the limitations of triggers and the bifurcation of business logic across application and database code spaces.  By encoding business logic at the application level the code is also significantly more portable assuming the use of an [ORM](https://en.wikipedia.org/wiki/Object-relational_mapping).

If you have multiple clients with direct access to the database, triggers are a useful tool to protect data contract expectations but do have limitations that are sometimes hard to work around.  Dependency on triggers and stores procedures also reduces portability and behavior transparency.

This article is not meant to be categorical by any stretch.  As will all things relating to software design your mileage will vary on your particular needs and requirements.
