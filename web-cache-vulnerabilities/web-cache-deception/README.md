# Web Cache Deception — PortSwigger Lab Write-Up

> **Category:** Web Cache Attacks | **Difficulty:** Practitioner  
> **Platform:** PortSwigger Web Security Academy

---

## 1. Overview

### What Is Web Cache Deception?

Web Cache Deception (WCD) is an attack that tricks a caching layer — a CDN, reverse proxy, or similar intermediary — into storing and serving a victim's private, authenticated response as if it were a public, cacheable static asset. The attacker then retrieves that cached response directly, bypassing any authentication entirely.

The reason this works is a fundamental architectural mismatch: **the origin server and the cache interpret the same URL differently**. Neither component is obviously broken in isolation. The origin correctly identifies and serves dynamic, user-specific content. The cache correctly stores responses it believes to be static. The problem only emerges at their intersection — which is exactly what makes WCD tricky to catch in code review.

### Why It Happens

Modern web stacks separate concerns: a backend application handles business logic, and a caching layer sits in front to improve performance. These two components may be configured independently, often by different teams, using different rules for determining what's cacheable.

Caches typically make caching decisions based on the file extension or path of the URL. If the URL looks like it ends in `.css`, `.js`, or `/image.png`, many caches will store the response. The origin server, however, may use its own routing logic that ignores or normalizes parts of the URL, serving dynamic content regardless of what the path looks like on the surface.

There are two primary mechanism classes:

**1. Static Extension-Based Bypass**  
An attacker appends a fake static extension (`/profile/wcd.js`) to a sensitive endpoint. The cache sees `.js` and caches the response. The origin sees `/profile` (or ignores the rest) and returns the authenticated user's data.

**2. URL Normalization Discrepancy**  
This is where things get more interesting — and more subtle. Consider the fragment character `#`. Both the origin and cache normally treat it as a delimiter, stripping everything after it. But if you URL-encode it:

```
/profile%23wcd.css
```

Now the behavior can diverge:
- **Origin server** normalizes `%23` back to `#` and interprets the path as `/profile` — serving the authenticated profile page.
- **Cache** does *not* normalize and sees the path as `/profile%23wcd.css` — caches it as a static CSS file.

A second normalization variant targets path traversal handling. When the cache *doesn't* normalize but the origin *does*, this payload can work:

```
/<static-directory-prefix>/..%2f<dynamic-path>
```

For example:
```
/static/..%2fapi/me
```

- The **cache** sees a request starting with `/static/` and caches it as a static asset.
- The **origin** normalizes `..%2f` into `../`, resolves the traversal, and routes the request to `/api/me` — returning authenticated user data.

### Why It's Dangerous

The danger is in how invisible the attack is. An attacker sends a crafted URL to a victim (via phishing, an embedded image, or any interaction that causes the victim's browser to make a request). The victim's authenticated browser follows the link. The response — including session data, personal details, tokens — gets cached publicly. The attacker just requests the same URL afterward and collects whatever the cache is holding. Nothing about this triggers an obvious alarm.

---

## 2. Lab Objective

The goal of this lab was to exploit a web cache deception vulnerability to steal another user's API key. The application caches static assets based on file extension, but the origin server's routing logic doesn't properly account for what the cache will do with certain URL structures. The task was to construct a URL that would cause the victim's authenticated response to be cached, then retrieve it.

---

## 3. Vulnerability Analysis

### The Misconfiguration

The root cause is a disconnect between two independently reasonable configurations:

- The **cache** is set up to store responses for URLs that look like static assets — specifically anything with a recognized extension like `.js`, `.css`, or `.png`.
- The **origin** routes requests based on its own path-matching logic, which either ignores the appended fake extension or normalizes encoded characters before routing — ultimately serving the same dynamic, authenticated content regardless of what's been tacked onto the URL.

### How Caching Behavior Was Abused

When the victim visits the crafted URL, the request hits the cache first. The cache has no stored response, so it forwards the request to the origin. The origin processes it — authenticated, serving user-specific data — and returns the response. The cache then stores that response, keyed against the full URL including the fake extension. The `Cache-Control` headers on the response either weren't set or were permissive enough that the cache didn't see a reason to skip storing it.

From that point on, any unauthenticated request to the same URL returns the cached response — no credentials required.

### Why Authentication Protections Failed

Authentication protected the origin, not the cache. Once a response is stored by the cache, subsequent requests for that URL never reach the origin at all. The cache doesn't re-authenticate. It just serves what it has. This means authentication is effectively bypassed for anyone who requests the URL after it's been cached — they're getting a stored copy of someone else's authenticated session output.

---

## 4. Exploitation Process

### Step 1 — Map Sensitive Endpoints

The first step was identifying which endpoints return user-specific, sensitive data — account details, API keys, profile information. These are the targets. Any endpoint that serves different content depending on who's logged in is a potential candidate.

### Step 2 — Understand the Cache's Rules

I needed to figure out what the cache considered cacheable. Testing a few requests with a known-cacheable static path confirmed the cache was operating on extension-matching. A request ending in `.js` would get stored; a plain dynamic route would not. That told me the attack surface.

### Step 3 — Construct the Payload URL

With the normalization behavior in mind, I constructed a URL that would appear static to the cache but route correctly to the sensitive endpoint on the origin. This meant appending a fake static suffix — or, in the normalization variant, using `%23` encoding or a path traversal pattern like `/static/..%2f<target-endpoint>`.

The key insight here is that the **split** happens between two systems:

- Cache sees: `/profile%23wcd.css` → "this looks like a CSS file, I'll cache it"
- Origin sees: `/profile` → "this is a valid route, here's the user's data"

Both are doing exactly what they're supposed to. The exploit lives in the gap between them.

### Step 4 — Deliver the URL to the Victim

The crafted URL needs to be requested by an authenticated user. In the lab this is simulated. In a real scenario it could be a phishing link, an injected resource, or any mechanism that causes the victim's browser to make the request while authenticated.

### Step 5 — Retrieve the Cached Response

After the victim's browser loads the URL, the response is stored in the cache. I then requested the same URL from an unauthenticated context. The cache returned the stored response — including the victim's API key — no login needed.

---

## 5. Real-World Impact

In a production environment, this class of vulnerability could expose:

- **Session tokens or API keys** — directly usable for account takeover
- **PII** — names, email addresses, billing details, government IDs
- **Internal application data** — anything rendered into the authenticated page response

The attack scales quietly. A single cache entry persists until TTL expiry, potentially serving leaked data to anyone who knows the URL during that window. Depending on cache TTL configuration, this could be minutes or hours. And because the attacker doesn't interact with the origin at all during retrieval, server-side logs may show nothing suspicious on their end.

For applications handling financial, medical, or identity data, a working WCD exploit could result in regulatory exposure (GDPR, HIPAA) in addition to direct user harm.

---

## 6. Mitigation Strategies

### Set Explicit Cache-Control Headers on Dynamic Routes

The most reliable mitigation is instructing the cache directly:

```http
Cache-Control: no-store, private
```

`no-store` tells caches not to persist the response at all. `private` signals it's intended only for the end user. Together, they make it clear to any compliant caching layer that this response must not be cached, regardless of what the URL looks like. These headers should be applied to every route that returns authenticated or user-specific content.

### Don't Rely on URL Shape for Cache Decisions

Caches should be configured to make caching decisions based on explicit metadata — `Cache-Control` headers — rather than inferring cacheability from file extensions or path patterns. Extension-based rules are fragile precisely because they can be spoofed by appending fake extensions.

### Normalize Consistently Across the Stack

If the origin normalizes encoded characters before routing, the cache should apply the same normalization before computing its cache key. A mismatch in how `%23`, `%2f`, or other encoded characters are handled is a direct enabler of normalization-based WCD. Either both components treat the encoded form the same way, or the cache key must be derived *after* normalization — not before.

### Scope Cache Keys Properly

CDNs and reverse proxies should be configured to include relevant request headers (like `Authorization` or `Cookie`) in cache keys for authenticated routes. This prevents a response generated for one user from being served to another. Most CDNs support this via cache key configuration.

### Validate URL Decoding in Path Traversal Scenarios

For the `/static/..%2f<target>` variant, the cache should refuse to serve requests where the resolved path escapes the intended static directory. Path traversal normalization should happen before cache key generation, not after.

---

## 7. Detection and Prevention

### For Defenders

**In code review:** Look for any route where the response content is user-specific but `Cache-Control` headers are missing, permissive, or inherited from a default that was designed for static assets.

**In traffic analysis:** Monitor for repeated requests to URLs with unusual extensions on dynamic paths — especially from unauthenticated sources. A pattern of `GET /api/user/profile.css` from a non-logged-in IP shortly after the same URL was hit by an authenticated user is a strong signal.

**In penetration testing:** Systematically append static extensions and encoded delimiters to authenticated endpoints and check whether the response is cached (look for `X-Cache: HIT`, `Age` headers, or repeated identical responses across unauthenticated sessions).

**With security headers auditing:** Automated scanning tools (Burp Suite, custom scripts) can check whether authenticated routes return appropriate `Cache-Control: no-store` directives. Any route missing this on user-specific content is worth investigating manually.

**In CDN/proxy configuration review:** Audit caching rules to confirm they don't override `Cache-Control` headers, and that cache key scoping is applied to sensitive routes. Misconfigured CDN "cache everything" rules are a common source of WCD in real deployments.

---

*Write-up authored as part of a personal web security research portfolio. Lab completed on PortSwigger Web Security Academy.*
