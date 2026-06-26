# JWT Authentication Bypass via JWK Header Injection

**Platform:** PortSwigger Web Academy  
**Category:** JWT Attacks  
**Difficulty:** Practitioner  
**Lab URL:** https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection  

---

## Objective

Access the `/admin` panel and delete the user `carlos` by injecting your own RSA public key into the JWT header so the server uses it to verify a token you signed yourself.

---

## Background

### What is a JWK?

A **JSON Web Key (JWK)** is a JSON object that represents a cryptographic key. The JWT specification allows an optional `jwk` parameter in the header — it's supposed to let a server know which key was used to sign the token (usually pointing to a trusted key set).

The vulnerability: some servers **fetch and use whatever public key is embedded in the `jwk` header**, without checking whether that key is actually trusted. 

This lets an attacker:
1. Generate their own RSA key pair
2. Sign a forged JWT with their **private key**
3. Embed their **public key** in the JWT header as `jwk`
4. The server uses the attacker's own public key to verify — and it passes

It's like a criminal signing a document, attaching their own notary stamp, and a judge accepting it as valid without checking if the notary is licensed.

---

## Tools Used

- Burp Suite (Proxy + Repeater)
- Burp JWT Editor extension (required — handles RSA key generation and signing)

---

## Steps

### 1. Log in and capture the JWT

Log in with:  
- Username: `wiener`  
- Password: `peter`

Send any authenticated request to Burp Repeater. You'll see the session JWT in the cookie.

---

### 2. Generate a new RSA key pair in Burp JWT Editor

1. Go to the **JWT Editor Keys** tab in Burp
2. Click **New RSA Key**
3. Keep default settings (2048-bit)
4. Click **Generate** → **OK**

This gives you a fresh RSA key pair stored in Burp. You'll use this to sign your forged token.

---

### 3. Modify the JWT payload

In Repeater, open the JWT tab. Change the `sub` field:

```json
{
  "iss": "portswigger",
  "sub": "administrator",
  "exp": 1720000000
}
```

---

### 4. Inject your public key into the header via `jwk`

In the JWT Editor tab in Repeater:

1. Click **Attack** → **Embedded JWK**
2. Select the RSA key you generated in Step 2
3. Click **OK**

Burp will:
- Add a `jwk` parameter to the JWT header containing your RSA public key
- Re-sign the entire token using your RSA **private key**

The resulting header looks like:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": {
    "kty": "RSA",
    "e": "AQAB",
    "n": "<your_public_modulus>",
    "kid": "<key_id>"
  }
}
```

The signature is now valid — but it's valid against **your key**, not the server's original key.

---

### 5. Send to `/admin` and delete carlos

Send the modified request with the forged JWT:

```
GET /admin HTTP/1.1
Cookie: session=<forged_token>
```

`200 OK` confirms the server verified the signature using your embedded public key. Then:

```
GET /admin/delete?username=carlos
```

Lab solved.

---

## Why This Works

The server is supposed to verify JWTs against a **trusted set of keys** it controls (typically fetched from its own JWKS endpoint at something like `/.well-known/jwks.json`). 

Instead, this server reads the `jwk` field from the token header and uses whatever public key it finds there to verify the signature — no trust check, no allowlist.

This is circular and meaningless: an attacker can always generate a key pair, sign their own token, and embed the matching public key. The server verifies the signature and it passes — but it proves nothing about the token's origin.

---

## How This Differs from the Previous Labs

| Lab | Core Vulnerability | Signature Status |
|---|---|---|
| Lab 01 — Unverified Signature | Server skips verification entirely | Ignored |
| Lab 02 — alg:none | Server trusts client-supplied algorithm choice | Removed |
| Lab 03 — JWK Injection | Server trusts client-supplied public key | Valid but self-signed |

Lab 03 is more sophisticated — the token actually **has a valid cryptographic signature**. The flaw isn't that verification is skipped; it's that the server verifies against an untrusted attacker-controlled key.

---

## Key Takeaway

> **Public keys must come from a trusted source the server controls — never from the token itself.** A JWT that carries its own verification key is trusting the attacker to define the rules of trust. Always validate JWTs against a hardcoded or server-fetched key set, and treat the `jwk` / `jku` / `x5u` header parameters as attacker-controlled input.

---

## Related Attacks (Same Family)

- **`jku` injection** — instead of embedding the key directly, attacker points `jku` to a URL they control that hosts a crafted JWKS
- **`x5u` injection** — same concept using X.509 certificate URLs
- **`kid` SQL/path injection** — manipulating the Key ID parameter to point to a null byte or predictable value

---
