# PortSwigger Lab: SSRF Attack Against the Local Network

**Platform:** PortSwigger Web Academy | **Category:** SSRF | **Difficulty:** Practitioner
**Tools:** Burp Suite (Proxy, Intruder)

## Objective
Use the stock check feature to scan the internal `192.168.0.X` range on port `8080` for an admin interface, then use it to delete the user `carlos`.

## Background
This lab extends basic SSRF from targeting the web server itself (`localhost`) to targeting a *different* machine on the internal network. Private IP addresses (like `192.168.0.X`) are not routable from the internet, so an external attacker cannot reach them directly. The web server, however, sits inside that network and can reach them. Since the stock check feature lets the client control the full backend URL, the web server can be used as a proxy to probe the internal network.

## Methodology
1. Intercepted the stock check request in Burp and confirmed the `stockApi` parameter accepts a full URL (`POST /product/stock`, body: `stockApi=<url>`).
2. Since the target machine's address was unknown, sent the request to **Intruder** and marked the last octet of the IP as the payload position: `stockApi=http://192.168.0.§0§:8080/admin`.
3. Configured a numeric payload list from 0 to 255.
4. Ran the attack and sorted results by status code / response length.
5. Identified one request returning `200 OK` with a distinct response length (3376 bytes) while all others returned `500` with a uniform length (2581 bytes) — the host with nothing listening failed consistently, the real host responded.
6. Loaded that response in the Inspector and found the admin panel HTML, including delete links for existing users, e.g. `/admin/delete?username=carlos`.
7. Sent a second request with `stockApi=http://192.168.0.92:8080/admin/delete?username=carlos` to delete the user. Lab solved.

## Payloads
```
# Step 1 - enumeration (Intruder payload position)
stockApi=http://192.168.0.§0-255§:8080/admin

# Step 2 - exploitation, using the discovered host
stockApi=http://192.168.0.92:8080/admin/delete?username=carlos
```

## Explanation
- This is **regular SSRF** (the response is returned to the attacker), executed in two stages: enumeration, then exploitation.
- The web server acted as a bridge into the private network: it could reach `192.168.0.92:8080` even though I, as an external attacker, could not reach it directly.
- The distinguishing signal between a live host and a dead one was the HTTP status code and response length — a listening service returned `200` with real content, while unreachable addresses returned a uniform `500` error.
- Intruder automated what would otherwise be 256 manual requests, by substituting each candidate octet into the same request template.

## Remediation
- **Do not accept a client-supplied destination URL.** The server should resolve stock information from a fixed, hardcoded internal address.
- If dynamic routing is required, enforce a strict **allowlist** of exact internal hosts/ports permitted, rejecting everything else by default.
- Internal admin interfaces should require authentication regardless of network origin; "reachable only internally" should never be treated as "trusted."
- Network segmentation / firewall rules so the web server's outbound reach is limited to only the services it legitimately needs.
- Rate-limit or detect high volumes of outbound requests with varying destinations from a single feature, which is a signature of this kind of scanning.

## Key Takeaways
- SSRF isn't limited to the local machine; it can pivot to any internal host the web server can route to.
- Private IP ranges are unreachable from the internet directly, but become reachable through a vulnerable server acting as a proxy.
- Differentiating "live" from "dead" internal hosts via status code / response length is a core technique for internal network enumeration via SSRF.
- Automating the scan with Intruder is the practical equivalent of manually trying all 256 addresses.

## References
- PortSwigger Web Academy: SSRF
- OWASP: Server-Side Request Forgery
