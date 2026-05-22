# HTTP Security Headers: A Practical Deep Dive

> **Author:** Ashwin Hari | Application Security Engineer  
> **Tags:** `web-security` `security-headers` `OWASP` `appsec`  
> **Last Updated:** May 2026

---

## Overview

Security headers are one of the most overlooked yet highest-impact controls in web application security. They're a single line of server configuration that can block entire classes of attack — XSS, clickjacking, MIME sniffing, and more.

This write-up covers the essential HTTP security headers, what they do, how to test for them, and where they fit in the OWASP Top 10.

---

## The Headers That Matter

### 1. `Content-Security-Policy` (CSP)

**What it does:** Tells the browser which sources of content (scripts, styles, images, fonts) are trusted. Blocks inline scripts and unauthorized third-party resources.

**Why it matters:** Mitigates Cross-Site Scripting (XSS) — OWASP A03:2021.

**Example — strict policy:**
```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none';
```

**Common misconfigurations:**
- `unsafe-inline` in `script-src` — completely defeats XSS protection
- Wildcard `*` in any directive — overly permissive
- Missing `object-src 'none'` — allows Flash/plugin injection
- Missing `base-uri 'none'` — allows base tag injection

---

### 2. `Strict-Transport-Security` (HSTS)

**What it does:** Forces browsers to use HTTPS for all future requests to the domain. Prevents SSL stripping attacks.

**Why it matters:** Mitigates man-in-the-middle attacks and protocol downgrade attacks.

**Example — recommended:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**Key parameters:**
| Parameter | Purpose |
|---|---|
| `max-age=31536000` | Cache HTTPS enforcement for 1 year |
| `includeSubDomains` | Apply to all subdomains |
| `preload` | Submit domain to browser preload lists |

**Common misconfigurations:**
- `max-age` set too low (e.g., `max-age=0` effectively disables it)
- Missing `includeSubDomains` — leaves subdomains exposed
- Applied on HTTP responses (only valid over HTTPS)

---

### 3. `X-Content-Type-Options`

**What it does:** Prevents browsers from MIME-sniffing a response away from the declared `Content-Type`.

**Why it matters:** Stops attacks where an attacker uploads a file (e.g., a `.jpg`) containing executable JavaScript that a browser might run.

**Example:**
```
X-Content-Type-Options: nosniff
```

Only one valid value: `nosniff`. Simple — always set it.

---

### 4. `X-Frame-Options`

**What it does:** Controls whether a page can be embedded in an `<iframe>`.

**Why it matters:** Prevents clickjacking — OWASP A05:2021 (Security Misconfiguration).

**Example:**
```
X-Frame-Options: DENY
```

| Value | Behaviour |
|---|---|
| `DENY` | Never render in a frame |
| `SAMEORIGIN` | Allow only same-origin frames |
| `ALLOW-FROM uri` | Deprecated — avoid |

> **Note:** CSP's `frame-ancestors` directive supersedes this header in modern browsers, but `X-Frame-Options` is still needed for legacy client support.

---

### 5. `Permissions-Policy`

**What it does:** Controls which browser features and APIs the page can access (camera, microphone, geolocation, payment, etc.).

**Why it matters:** Reduces attack surface if the app is ever compromised or includes malicious third-party scripts.

**Example — restrictive:**
```
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()
```

---

### 6. `Referrer-Policy`

**What it does:** Controls how much referrer information is included in requests leaving your site.

**Why it matters:** Prevents leaking sensitive URL parameters (tokens, session IDs) to third parties.

**Example:**
```
Referrer-Policy: strict-origin-when-cross-origin
```

---

### 7. `Cache-Control`

**What it does:** Instructs browsers and proxies on how to cache the response.

**Why it matters:** Prevents sensitive data (authenticated pages, financial data, PII) from being stored in browser or proxy caches.

**Example — for sensitive pages:**
```
Cache-Control: no-store, no-cache, must-revalidate
```

---

## Testing for Missing Headers

### Manual — curl
```bash
curl -I https://target.com
```

### Python — quick scanner snippet
```python
import requests

HEADERS_TO_CHECK = [
    "Content-Security-Policy",
    "Strict-Transport-Security",
    "X-Content-Type-Options",
    "X-Frame-Options",
    "Permissions-Policy",
    "Referrer-Policy",
]

def check_headers(url):
    resp = requests.get(url, timeout=10)
    print(f"\n[+] Checking: {url}\n")
    for header in HEADERS_TO_CHECK:
        if header in resp.headers:
            print(f"  ✅  {header}: {resp.headers[header]}")
        else:
            print(f"  ❌  MISSING: {header}")

check_headers("https://example.com")
```

> This is the kind of check baked into the [`web-security-scanner`](https://github.com/ashwin-tech-sec/web-security-scanner) tool in this org.

### Tools
- **OWASP ZAP** — Passive scan flags missing headers
- **Burp Suite** — Passive scanner covers CSP, HSTS, X-Frame-Options
- **securityheaders.com** — Quick online scan with grading

---

## OWASP Mapping

| Header | OWASP Category |
|---|---|
| Content-Security-Policy | A03: Injection (XSS) |
| Strict-Transport-Security | A02: Cryptographic Failures |
| X-Frame-Options | A05: Security Misconfiguration |
| X-Content-Type-Options | A05: Security Misconfiguration |
| Permissions-Policy | A05: Security Misconfiguration |

---

## Quick Reference — Recommended Header Set

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none';
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Permissions-Policy: camera=(), microphone=(), geolocation=()
Referrer-Policy: strict-origin-when-cross-origin
Cache-Control: no-store
```

---

## References

- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [MDN HTTP Headers Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)
- [securityheaders.com](https://securityheaders.com)
- [Content Security Policy Reference](https://content-security-policy.com/)
- [OWASP Top 10 — A05: Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)

---

*Part of a series of cybersecurity write-ups by [@ashwin-tech-sec](https://github.com/ashwin-tech-sec)*
