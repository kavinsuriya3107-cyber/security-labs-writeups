Lab 01: JWT Authentication Bypass via Unverified Signature

Platform: PortSwigger Web Security Academy

Category: JWT

Difficulty: Apprentice

Status: ✅ Solved

Tool Used: Burp Suite Community Edition v2026.3.2 + JWT Editor Extension


📌 Lab Description

This lab uses a JWT-based session mechanism. The server accepts any JWT submitted without verifying its signature. 
The goal is to exploit this misconfiguration to access the /admin panel by modifying the JWT payload to impersonate the administrator user,
then delete the user carlos.


🔍 Vulnerability Explanation

JSON Web Tokens (JWTs) consist of three parts:

Header.Payload.Signature

The signature is supposed to cryptographically verify that the token wasn't tampered with.
However, some flawed implementations:


Decode and use the payload directly
Never validate the signature against the secret key


This means an attacker can freely modify the payload (e.g., change "sub": "wiener" to "sub": "administrator") and the server will accept it as legitimate.


🛠️ Steps to Exploit

Step 1: Login as Normal User


Login with credentials: wiener:peter
Intercept the POST /login request in Burp Suite
Observe the session cookie — it is a JWT token


Step 2: Inspect the JWT


Send the request to Repeater
Click the JSON Web Token tab in Burp (JWT Editor extension)
Observe the JWT structure:


Header:

json{
  "typ": "JWT",
  "alg": "RS256"
}

Payload:

json{
  "iss": "portswigger",
  "exp": 1780850461,
  "sub": "wiener"
}

Step 3: Modify the Payload


In the JWT Editor Payload section, change:


json"sub": "wiener"

to:

json"sub": "administrator"

Step 4: Forward the Modified Request


Do NOT re-sign the token — leave the original signature unchanged
Send the request with the tampered JWT to /my-account?id=administrator
The server accepts the token without verifying the signature
Response: HTTP 200 OK — admin panel accessible


Step 5: Delete User Carlos


Navigate to /admin
Delete the user carlos
Lab marked as Solved ✅



🔬 Why This Works

The server decodes the JWT payload and uses the sub claim to identify the user.
It never checks whether the signature matches the payload. 
Since we only changed the payload and kept the old (now invalid) signature, 
the server still accepts it — proving the signature verification step is completely absent.
