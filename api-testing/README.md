# API Testing  Web Security Academy Labs

> Completed hands-on labs covering API recon, endpoint discovery, and mass assignment exploitation.

---

## What is API Testing?
APIs are the backbone of every dynamic website. Testing them means looking for logic flaws, improper access controls, and parameters that developers never intended to be user-controlled. Vulnerabilities here can undermine confidentiality, integrity, and availability at the core level.

---

## Recon  Mapping the Attack Surface

Before touching anything, I map out what the API exposes:

- **Find endpoints**  look for `/api/`, `/swagger/index.html`, `/openapi.json` in URLs and JS files
- **Check JS files**  they often reference endpoints never triggered by the UI
- **Investigate base paths**  if you find `/api/swagger/v1/users/123`, walk back to `/api/swagger/v1`, `/api/swagger`, `/api`
- **What to document:** supported HTTP methods, input params (required + optional), auth mechanisms, rate limits

---

## Endpoint Interaction

API endpoints often behave differently depending on how you talk to them.

**Test all HTTP methods**  an endpoint might accept GET but also a hidden DELETE or PATCH:
```
GET    /api/tasks  →  retrieve tasks
POST   /api/tasks  →  create task
DELETE /api/tasks/1  →  delete task
```

**Change the Content-Type**  switching from JSON to XML can bypass defenses or expose injection points the developer never accounted for.

**Use Burp Intruder** with wordlists to brute-force hidden endpoints (e.g. found `/update` → try `/delete`, `/add`, `/admin`).

---

## Finding Hidden Parameters

Developers sometimes leave internal object fields exposed without realizing they're writable.

- **Param Miner BApp**  automatically guesses up to 65,536 param names per request
- **Compare responses**  if a PATCH only accepts `username` + `email`, but a GET returns `id`, `name`, `isAdmin`  those extra fields may be writable too

---

## Mass Assignment (Lab Completed ✓)

**What it is:** Frameworks automatically bind incoming request fields to internal object fields. If the server doesn't restrict which fields can be set, an attacker can overwrite sensitive ones.

**Real example from the lab:**

The API schema exposed a `chosen_discount` object with a `percentage` field that defaulted to `0`. Since the server performed no allowlist validation, I was able to inject it directly into the POST request:

```json
{
  "chosen_products": [...],
  "chosen_discount": { "percentage": 100 }
}
```

This applied a 100% discount  a field never meant to be user-controlled.

**Key insight:** `"default: 0"` is NOT sanitization. It's just an initialization value for when the field is omitted at object creation. If you explicitly send a value, the default is bypassed entirely. The GET endpoint simply reads what's stored  defaults are never re-applied on read.

**How to test mass assignment:**
1. Identify hidden fields by comparing what you send vs what the API returns
2. Send an **invalid value** first  if behavior changes, the field is being processed
3. Then send a **malicious value** (e.g. `"isAdmin": true`, `"percentage": 100`)

---

## Prevention

| Issue | Fix |
|-------|-----|
| Mass assignment | Allowlist writable fields + blocklist sensitive ones |
| Hidden endpoints | Apply auth + rate limits to ALL API versions |
| Info leakage | Use generic error messages |
| Content type abuse | Validate `Content-Type` on every request |
| Exposed docs | Restrict access if not public-facing |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Burp Suite (Repeater + Intruder) | Interact with and fuzz endpoints |
| Param Miner BApp | Discover hidden/undocumented parameters |
| Burp Scanner | Crawl app and audit OpenAPI documentation |
