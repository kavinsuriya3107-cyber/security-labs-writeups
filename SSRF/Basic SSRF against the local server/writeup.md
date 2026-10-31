# PortSwigger Lab: Basic SSRF against the local server

**Platform:** PortSwigger Web Academy | **Category:** SSRF | **Difficulty:** Apprentice
**Tools:** Burp Suite (Proxy, Repeater)

## Objective
Abuse the stock check feature to reach the internal admin interface at `http://localhost/admin` and delete the user `carlos`.

## Background
Server-Side Request Forgery (SSRF) occurs when an attacker makes the server send requests to a destination the attacker chooses. The server sits inside the network and is trusted by internal services, so it can reach resources that are not exposed externally. In this lab, the application fetches a stock-check URL supplied by the client, with no validation.

## Methodology
1. Browsed the shop and identified the "Check stock" feature.
2. Intercepted the stock check request in Burp and found the `stockApi` parameter containing a full backend URL.
3. Concluded the server fetches whatever URL the client provides (client-controlled destination).
4. Replaced the value with the internal admin URL and observed the admin panel returned in the response (regular SSRF, response visible).
5. Identified the delete endpoint for users in the admin panel.
6. Sent the request to the delete endpoint for `carlos`. Lab solved.

## Payloads
```
stockApi=http://localhost/admin
stockApi=http://localhost/admin/delete?username=carlos
```
(In the actual request the value is URL-encoded.)

## Explanation
- The server made the request itself, from inside the network, so the internal-only admin panel treated it as trusted local traffic.
- No validation existed on `stockApi`, so no filter bypass was needed.
- This is **regular SSRF**: the fetched response was returned to me.

## Remediation
- **Do not accept a destination URL from the client.** Accept only a product/store ID and build the backend URL server-side from a hardcoded trusted base.
- If a URL parameter is unavoidable: strict **allowlist** of permitted hosts (not a blocklist), and validate the **resolved IP** at connection time.
- Do not rely on "internal means trusted": require authentication on admin interfaces even from localhost.
- Network segmentation so the web server cannot reach sensitive internal services.

## Key Takeaways
- Any feature where the server fetches a user-supplied URL is an SSRF candidate.
- The root cause is trusting client input for the request destination.
- Blocklisting `localhost` / `127.0.0.1` is a weak fix; the next labs cover bypasses.

## References
- PortSwigger Web Academy: SSRF
- OWASP: Server-Side Request Forgery
