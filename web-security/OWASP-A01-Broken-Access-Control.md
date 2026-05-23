# OWASP A01: Broken Access Control

> **Author:** Ashwin Hari | Application Security Engineer  
> **Tags:** `OWASP` `access-control` `web-security` `appsec` `authorisation`  
> **Last Updated:** May 2026

---

## Overview

Broken Access Control has ranked **#1 in the OWASP Top 10 since 2021**, making it the most widespread and impactful web application vulnerability category. It occurs when users can act outside of their intended permissions, accessing data or functionality that should be restricted.

This write-up covers what broken access control looks like in practice, the most common vulnerability patterns, how to test for them, and how to fix them.

---

## What Is Access Control?

Access control enforces that users can only perform actions and access resources they are authorised to. It sits on top of authentication, authentication confirms *who you are*, access control determines *what you can do*.

There are three main types:

| Type | Description | Example |
|---|---|---|
| **Vertical access control** | Restricts access by privilege level | Regular user accessing admin functionality |
| **Horizontal access control** | Restricts access to same-privilege resources | User A accessing User B's data |
| **Context-dependent access control** | Restricts access based on application state | Editing an order after it has been paid |

---

## Common Vulnerability Patterns

### 1. Insecure Direct Object Reference (IDOR)

One of the most common and impactful access control flaws. The application uses user-controlled input to reference objects directly without verifying the requesting user has permission to access them.

**Example**

**vulnerable URL:**
```
GET /api/invoices/1042
```

An attacker simply changes the ID:
```
GET /api/invoices/1043
GET /api/invoices/1044
```

If the server returns another user's invoice without checking ownership, that's IDOR.

**Vulnerable code (Python/Flask):**
```python
@app.route('/api/invoices/<int:invoice_id>')
def get_invoice(invoice_id):
    invoice = Invoice.query.get(invoice_id)
    return jsonify(invoice)  # No ownership check!
```

**Fixed code:**
```python
@app.route('/api/invoices/<int:invoice_id>')
@login_required
def get_invoice(invoice_id):
    invoice = Invoice.query.get(invoice_id)
    if invoice.user_id != current_user.id:
        abort(403)
    return jsonify(invoice)
```

---

### 2. Missing Function Level Access Control

The application exposes administrative or privileged functionality without enforcing authorisation checks, often relying on the UI hiding the link rather than the server enforcing the restriction.

**Example:**
```
GET /admin/users          → 200 OK (no auth check on the server)
POST /admin/users/delete  → 200 OK (any logged-in user can call this)
```

The admin panel link may not appear in the UI for regular users, but the endpoints themselves are wide open.

**Key point:** Security by obscurity is not access control. If the server doesn't enforce it, the UI hiding it means nothing.

---

### 3. Privilege Escalation

**Vertical privilege escalation** — a regular user gains admin-level access.

**Example — vulnerable request:**
```http
POST /api/user/update
{
  "user_id": 99,
  "role": "admin"
}
```

If the server accepts a `role` parameter from the client without validating the requesting user's own role, any user can promote themselves to admin.

**Horizontal privilege escalation** — a user accesses another user's data at the same privilege level. This is essentially IDOR at scale.

---

### 4. JWT / Token Manipulation

JSON Web Tokens that aren't properly validated can be tampered with to escalate privileges.

**Example**

**decoded JWT payload (before tampering):**
```json
{
  "user_id": 99,
  "role": "user"
}
```

**After tampering:**
```json
{
  "user_id": 99,
  "role": "admin"
}
```

If the server doesn't properly verify the JWT signature, or worse, accepts `alg: none`, this manipulation succeeds.

---

### 5. Path Traversal

Attackers manipulate file paths to access files outside the intended directory.

**Example:**
```
GET /download?file=../../etc/passwd
```

If the server doesn't sanitise the `file` parameter, this returns the system's password file.

---

### 6. CORS Misconfiguration

A misconfigured Cross-Origin Resource Sharing policy can allow an attacker's origin to make authenticated requests to the API.

**Vulnerable server response:**
```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

Allowing credentials with a wildcard origin is a critical misconfiguration, it allows any website to make authenticated cross-origin requests on behalf of a logged-in user.

---

## Testing Methodology

### 1. Map all endpoints and parameters
Identify every endpoint the application exposes, including undocumented ones. Check for:
- IDs in URLs (`/users/123`, `/orders/456`)
- IDs in request bodies
- Role or privilege parameters

### 2. Test horizontal access control (IDOR)
- Log in as User A, note resource IDs
- Log in as User B, attempt to access User A's resources using the same IDs
- Automate with Burp Suite's Intruder or a simple Python script

### 3. Test vertical access control
- Log in as a low-privilege user
- Attempt to access admin endpoints directly
- Replay admin requests with a regular user's session token

### 4. Check JWT handling
- Decode the JWT (base64) and inspect the payload
- Test `alg: none`, set algorithm to none and remove the signature
- Attempt to modify role/privilege claims

### Python — basic IDOR tester
```python
import requests

BASE_URL = "https://target.com/api/invoices"
SESSION_COOKIE = "your_session_cookie_here"

for invoice_id in range(1000, 1100):
    response = requests.get(
        f"{BASE_URL}/{invoice_id}",
        cookies={"session": SESSION_COOKIE}
    )
    if response.status_code == 200:
        print(f"[!] Accessible: /api/invoices/{invoice_id}")
```

> Always run this only on applications you are authorised to test.

---

## Remediation

| Issue | Fix |
|---|---|
| IDOR | Always verify object ownership server-side before returning data |
| Missing function access control | Enforce authorisation checks on every endpoint server-side — not just in the UI |
| Privilege escalation | Never trust client-supplied role or privilege parameters |
| JWT tampering | Validate JWT signatures server-side; reject `alg: none`; use strong secrets |
| Path traversal | Validate and sanitise file path inputs; use allow-lists for permitted paths |
| CORS misconfiguration | Never combine `Allow-Credentials: true` with wildcard origin |

**General principles:**
- Default deny — deny all access unless explicitly granted
- Enforce access control server-side on every request
- Log access control failures — they are often the first indicator of an attack in progress
- Use tested, centralised access control libraries rather than rolling your own

---

## Hands-on Demo

To see IDOR in action and test the fix yourself, check out the accompanying tool and vulnerable app:

- [idor-tester](https://github.com/ashwin-tech-sec/idor-tester) — Python scanner + intentionally vulnerable Flask app

Clone it locally, start the vulnerable app, and run the tester against both the `/vulnerable` and `/secure` endpoints to see the difference ownership checks make.

---

## OWASP References

- [OWASP A01:2021 — Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [OWASP Testing Guide — Testing for IDOR](https://owasp.org/www-project-web-security-testing-guide/)
- [OWASP Access Control Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html)
- [PortSwigger Web Academy — Access Control](https://portswigger.net/web-security/access-control)

---

*Part of a series of cybersecurity write-ups by [@ashwin-tech-sec](https://github.com/ashwin-tech-sec)*
