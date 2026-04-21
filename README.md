==================== SQL FUNDAMENTALS (OSCP NOTES) ====================

1. What is SQL?
- SQL (Structured Query Language) is used to interact with databases.
- It is used to:
  • Retrieve data
  • Insert data
  • Update data
  • Delete data
  • Create databases and tables
  • Manage users and permissions

-----------------------------------------------------------------------

2. MySQL Login (Command Line)

Local login:
mysql -u root -p

Remote login:
mysql -h <IP> -P <PORT> -u <USER> -p

Notes:
- -u = username
- -p = password (prompt)
- -P = port (default is 3306)

-----------------------------------------------------------------------

3. Basic Database Commands

SHOW DATABASES;
USE database_name;
CREATE DATABASE users;

Notes:
- SQL is case-insensitive
- Database names are case-sensitive

-----------------------------------------------------------------------

4. Tables Concept

- Data is stored in tables (rows and columns)
- Row = record
- Column = field

Create table:
CREATE TABLE logins (
    id INT,
    username VARCHAR(100),
    password VARCHAR(100),
    date_of_joining DATETIME
);

-----------------------------------------------------------------------

5. Table Constraints

id INT NOT NULL AUTO_INCREMENT,
username VARCHAR(100) UNIQUE NOT NULL,
password VARCHAR(100) NOT NULL,
date_of_joining DATETIME DEFAULT NOW(),
PRIMARY KEY (id)

Meaning:
- NOT NULL → cannot be empty
- UNIQUE → no duplicates allowed
- AUTO_INCREMENT → auto increases value
- DEFAULT → default value assigned
- PRIMARY KEY → unique identifier

-----------------------------------------------------------------------

6. Table Information

SHOW TABLES;
DESCRIBE table_name;

-----------------------------------------------------------------------

7. INSERT (Add Data)

INSERT INTO logins VALUES(1, 'admin', 'pass', '2020-07-02');

Insert specific columns:
INSERT INTO logins(username, password) VALUES('admin', '123');

Insert multiple rows:
INSERT INTO logins(username, password)
VALUES ('john','123'), ('tom','456');

Note:
- NOT NULL columns must be filled

-----------------------------------------------------------------------

8. SELECT (Retrieve Data)

SELECT * FROM logins;
SELECT username, password FROM logins;

- * means all columns

-----------------------------------------------------------------------

9. WHERE (Filtering Data)

SELECT * FROM logins WHERE id > 1;
SELECT * FROM logins WHERE username = 'admin';

Notes:
- Strings use quotes
- Numbers do not need quotes

-----------------------------------------------------------------------

10. LIKE (Pattern Matching)

SELECT * FROM logins WHERE username LIKE 'admin%';
SELECT * FROM logins WHERE username LIKE '___';

Symbols:
- % → multiple characters
- _ → single character

-----------------------------------------------------------------------

11. ORDER BY (Sorting)

SELECT * FROM logins ORDER BY password;
SELECT * FROM logins ORDER BY password DESC;

- Default sorting = ASC (ascending)

-----------------------------------------------------------------------

12. LIMIT (Restrict Output)

SELECT * FROM logins LIMIT 2;
SELECT * FROM logins LIMIT 1,2;

- Offset starts from 0

-----------------------------------------------------------------------

13. UPDATE (Modify Data)

UPDATE logins SET password='newpass' WHERE id > 1;

Warning:
- Without WHERE → all records will be updated

-----------------------------------------------------------------------

14. ALTER (Modify Table)

ALTER TABLE logins ADD newColumn INT;
ALTER TABLE logins RENAME COLUMN old TO new;
ALTER TABLE logins MODIFY column DATE;
ALTER TABLE logins DROP column;

-----------------------------------------------------------------------

15. DROP (Delete Table)

DROP TABLE logins;

Warning:
- Permanent deletion (no recovery)

-----------------------------------------------------------------------

16. Logical Operators

AND:
SELECT * FROM logins WHERE id > 1 AND username='admin';

OR:
SELECT * FROM logins WHERE id=1 OR username='admin';

NOT:
SELECT * FROM logins WHERE NOT username='john';

Notes:
- TRUE = 1
- FALSE = 0

-----------------------------------------------------------------------

17. Symbol Operators

AND → &&
OR  → ||
NOT → !

-----------------------------------------------------------------------

18. Operator Precedence (Important)

Order of execution:
1. * / %
2. + -
3. = > < != LIKE
4. NOT
5. AND
6. OR

-----------------------------------------------------------------------

19. Important for SQL Injection (OSCP)

- SELECT is most important
- WHERE clause is main injection point
- Common payload:
  ' OR 1=1 --

- Useful for:
  • Login bypass
  • Data extraction
  • Enumeration

-----------------------------------------------------------------------

20. Quick Revision

CRUD:
- Create → INSERT
- Read   → SELECT
- Update → UPDATE
- Delete → DELETE / DROP

Other:
- Filter → WHERE
- Sort → ORDER BY
- Limit → LIMIT
- Logic → AND / OR / NOT

=======================================================================
