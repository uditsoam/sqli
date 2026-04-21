# SQL Fundamentals 
##  Introduction

SQL (Structured Query Language) is used to interact with relational databases like MySQL.
It allows you to:

* Retrieve data
* Insert data
* Update data
* Delete data
* Create databases and tables
* Manage users and permissions

---

##  MySQL Login

```bash
mysql -u root -p
mysql -h <IP> -P <PORT> -u <USER> -p
```

* -u → username
* -p → password prompt
* -P → port (default: 3306)

---

##  Database Commands

```sql
SHOW DATABASES;
USE database_name;
CREATE DATABASE users;
```

* SQL is case-insensitive
* Database names are case-sensitive

---

##  Tables

* Data is stored in tables (rows and columns)
* Row = record
* Column = field

```sql
CREATE TABLE logins (
    id INT,
    username VARCHAR(100),
    password VARCHAR(100),
    date_of_joining DATETIME
);
```

---

##  Table Constraints

```sql
id INT NOT NULL AUTO_INCREMENT,
username VARCHAR(100) UNIQUE NOT NULL,
password VARCHAR(100) NOT NULL,
date_of_joining DATETIME DEFAULT NOW(),
PRIMARY KEY (id)
```

* NOT NULL → cannot be empty
* UNIQUE → no duplicate values
* AUTO_INCREMENT → auto increase
* DEFAULT → default value
* PRIMARY KEY → unique identifier

---

##  Table Info

```sql
SHOW TABLES;
DESCRIBE table_name;
```

---

##  INSERT (Add Data)

```sql
INSERT INTO logins VALUES(1, 'admin', 'pass', '2020-07-02');
INSERT INTO logins(username, password) VALUES('admin', '123');
INSERT INTO logins(username, password) VALUES ('john','123'), ('tom','456');
```

---

##  SELECT (Retrieve Data)

```sql
SELECT * FROM logins;
SELECT username, password FROM logins;
```

* * = all columns

---

##  WHERE (Filtering)

```sql
SELECT * FROM logins WHERE id > 1;
SELECT * FROM logins WHERE username = 'admin';
```

* Strings → use quotes
* Numbers → no quotes

---

## LIKE (Pattern Matching)

```sql
SELECT * FROM logins WHERE username LIKE 'admin%';
SELECT * FROM logins WHERE username LIKE '___';
```

* % → multiple characters
* _ → single character

---

##  ORDER BY (Sorting)

```sql
SELECT * FROM logins ORDER BY password;
SELECT * FROM logins ORDER BY password DESC;
```

* Default = ASC

---

##  LIMIT (Restrict Output)

```sql
SELECT * FROM logins LIMIT 2;
SELECT * FROM logins LIMIT 1,2;
```

* Offset starts from 0

---

##  UPDATE (Modify Data)

```sql
UPDATE logins SET password='newpass' WHERE id > 1;
```

 Without WHERE → all rows updated

---

## 🛠️ ALTER (Modify Table)

```sql
ALTER TABLE logins ADD newColumn INT;
ALTER TABLE logins RENAME COLUMN old TO new;
ALTER TABLE logins MODIFY column DATE;
ALTER TABLE logins DROP column;
```

---

##  DROP (Delete Table)

```sql
DROP TABLE logins;
```

 Permanent deletion

---

##  Logical Operators

```sql
SELECT * FROM logins WHERE id > 1 AND username='admin';
SELECT * FROM logins WHERE id=1 OR username='admin';
SELECT * FROM logins WHERE NOT username='john';
```

* TRUE = 1
* FALSE = 0

---

##  Symbol Operators

* AND → &&
* OR → ||
* NOT → !

---

##  Operator Precedence

1. * / %
2. * -
3. = > < != LIKE
4. NOT
5. AND
6. OR

---

##  SQL Injection Basics (Important)

* SELECT is most important
* WHERE clause = main injection point

```sql
' OR 1=1 --
```

Used for:

* Login bypass
* Data extraction
* Enumeration

---

##  Quick Revision

CRUD:

* Create → INSERT
* Read → SELECT
* Update → UPDATE
* Delete → DELETE / DROP

Other:

* Filter → WHERE
* Sort → ORDER BY
* Limit → LIMIT
* Logic → AND / OR / NOT

---
