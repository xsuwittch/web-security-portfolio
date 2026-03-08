
# API Testing Notes

## Recon
- Find endpoints: look for `/api/`, `/swagger/index.html`, `/openapi.json` in URLs and JS files
- Discover: supported methods, input params, auth mechanisms, rate limits

## Endpoints & Methods
- Test ALL HTTP methods per endpoint (GET, POST, PATCH, DELETE, OPTIONS)
- Change `Content-Type` header — may bypass defenses or expose different behavior
- Use Burp Intruder + wordlists to brute-force hidden endpoints

## Hidden Parameters
- **Param Miner** — guesses up to 65k param names
- Compare PATCH/POST fields vs GET response — extra fields in GET may be writable

## Mass Assignment
Framework auto-binds request fields to internal object fields — attacker can modify fields never meant to be user-controlled.

**How to test:**
1. Send invalid value → check if behavior changes
2. If yes, send malicious value (e.g. `"isAdmin": true`)

> `"default: 0"` is NOT a security control — it only fills missing fields at creation, it does NOT block values you explicitly send.

## Prevention
- Allowlist writable fields, blocklist sensitive ones
- Validate content type per request
- Apply protections to ALL API versions
- Use generic error messages

## Tools
| Tool | Use |
||-|
| Burp Scanner | Crawl & find endpoints |
| Burp Intruder | Brute-force methods/params |
| Param Miner | Discover hidden params |
| JS Link Finder | Extract endpoints from JS |
| Content Type Converter | Switch JSON ↔ XML |
