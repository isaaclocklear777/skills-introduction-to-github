# Web Application Security Best Practices

A practical guide to understanding and preventing common web application vulnerabilities.

---

## Table of Contents

1. [SQL Injection](#1-sql-injection)
2. [Cross-Site Scripting (XSS)](#2-cross-site-scripting-xss)
3. [Cross-Site Request Forgery (CSRF)](#3-cross-site-request-forgery-csrf)
4. [Insecure Direct Object References (IDOR)](#4-insecure-direct-object-references-idor)
5. [Security Misconfiguration](#5-security-misconfiguration)
6. [Sensitive Data Exposure](#6-sensitive-data-exposure)
7. [Authentication & Session Management](#7-authentication--session-management)
8. [General Best Practices](#8-general-best-practices)
9. [Additional Resources](#9-additional-resources)

---

## 1. SQL Injection

### What Is It?
SQL injection occurs when an attacker inserts or "injects" malicious SQL code into an input field that is later executed by the database. This can allow an attacker to read, modify, or delete data they should not have access to.

### Example Scenario
A login form accepts a username and password. If the application builds a SQL query by directly concatenating user input:

```sql
-- Vulnerable query
SELECT * FROM users WHERE username = 'admin' AND password = '' OR '1'='1';
```

The injected `' OR '1'='1'` always evaluates to true, potentially bypassing authentication.

### How to Prevent It

- **Use parameterized queries (prepared statements)** — Never concatenate user input directly into SQL strings.
  ```python
  # Safe example (Python with sqlite3)
  cursor.execute("SELECT * FROM users WHERE username = ? AND password = ?", (username, password))
  ```
- **Use an ORM** — Object-relational mappers (e.g., SQLAlchemy, Hibernate, ActiveRecord) handle parameterization automatically.
- **Validate and sanitize input** — Reject inputs that don't match expected formats (e.g., reject non-alphanumeric characters in a username field).
- **Apply the principle of least privilege** — Database accounts used by the application should have only the permissions they need (e.g., no `DROP TABLE` access for a read-only user).
- **Enable error handling** — Never expose raw database error messages to end users.

---

## 2. Cross-Site Scripting (XSS)

### What Is It?
XSS occurs when an attacker injects malicious scripts into content that is then served to and executed by other users' browsers. It can be used to steal session cookies, redirect users, or deface pages.

There are three main types:
- **Stored XSS** — Malicious script is saved in the database and served to all users who view that content.
- **Reflected XSS** — Malicious script is part of the request URL and immediately reflected back in the response.
- **DOM-based XSS** — The script is executed via client-side JavaScript manipulating the DOM.

### How to Prevent It

- **Output encoding** — Encode all user-supplied data before rendering it in HTML. Most modern frameworks do this automatically.
  ```html
  <!-- Unsafe -->
  <p>Hello, <%= userInput %></p>

  <!-- Safe (encoded output) -->
  <p>Hello, <%= h(userInput) %></p>
  ```
- **Use a Content Security Policy (CSP)** — A CSP header tells the browser which sources of scripts are trusted, blocking inline scripts and untrusted sources.
  ```
  Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted-cdn.example.com
  ```
- **Avoid `innerHTML` and `eval()`** — Use safer DOM APIs like `textContent` instead of `innerHTML` when inserting user data.
- **Validate input on the server** — Reject or sanitize inputs containing unexpected HTML or script tags.
- **Use HTTPOnly cookies** — Mark session cookies as `HttpOnly` so they cannot be accessed via JavaScript, limiting the damage of XSS attacks.

---

## 3. Cross-Site Request Forgery (CSRF)

### What Is It?
CSRF tricks an authenticated user's browser into sending an unintended request to a web application where the user is logged in. Because the browser automatically includes cookies, the server may accept the forged request as legitimate.

### Example Scenario
A user is logged into their bank. An attacker hosts a page with a hidden form that submits a fund transfer to the bank's endpoint. When the user visits the attacker's page, the form is submitted automatically using the user's session.

### How to Prevent It

- **Use CSRF tokens** — Include a unique, unpredictable token in every state-changing form submission. Validate it on the server.
  ```html
  <form method="POST" action="/transfer">
    <input type="hidden" name="csrf_token" value="{{ csrf_token }}">
    <!-- other fields -->
  </form>
  ```
- **Use the `SameSite` cookie attribute** — Setting `SameSite=Strict` or `SameSite=Lax` on cookies prevents them from being sent with cross-site requests.
  ```
  Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly
  ```
- **Check the `Origin` and `Referer` headers** — Reject requests where these headers indicate a cross-origin source.
- **Require re-authentication for sensitive actions** — For high-risk operations (e.g., changing a password or email), ask the user to confirm their password again.

---

## 4. Insecure Direct Object References (IDOR)

### What Is It?
IDOR occurs when an application exposes internal implementation objects (like database record IDs) directly to users without proper access control checks. An attacker can manipulate these references to access unauthorized data.

### Example Scenario
A URL like `/invoices/1042` lets a user view their invoice. Changing the ID to `/invoices/1043` reveals another user's invoice if no authorization check is performed.

### How to Prevent It

- **Always verify authorization server-side** — Check that the currently authenticated user owns or has permission to access the requested resource.
- **Use indirect references** — Replace predictable IDs with opaque tokens (e.g., UUIDs) that are harder to guess, but do not rely on this alone.
- **Implement role-based access control (RBAC)** — Define clear rules for who can access what, and enforce them consistently.

---

## 5. Security Misconfiguration

### What Is It?
Security misconfiguration is one of the most common vulnerabilities and includes things like default credentials, verbose error messages, unnecessary features enabled, or missing security headers.

### How to Prevent It

- **Disable or remove unused features, endpoints, and services.**
- **Change all default credentials** — Never deploy with default usernames and passwords.
- **Keep software up to date** — Regularly patch frameworks, libraries, and server software.
- **Set appropriate HTTP security headers:**

| Header | Purpose |
|--------|---------|
| `Strict-Transport-Security` | Enforces HTTPS connections |
| `X-Content-Type-Options: nosniff` | Prevents MIME-type sniffing |
| `X-Frame-Options: DENY` | Prevents clickjacking |
| `Content-Security-Policy` | Controls permitted content sources |
| `Referrer-Policy` | Controls referrer information |

- **Disable detailed error messages in production** — Show generic error pages to users; log details server-side.

---

## 6. Sensitive Data Exposure

### What Is It?
Applications that do not adequately protect sensitive data (passwords, payment information, personal data) can expose it through insecure transmission, weak encryption, or improper storage.

### How to Prevent It

- **Use HTTPS everywhere** — Enforce TLS for all connections; redirect HTTP to HTTPS.
- **Hash passwords properly** — Use a strong, slow hashing algorithm like **bcrypt**, **scrypt**, or **Argon2**. Never store plaintext or MD5/SHA1-hashed passwords.
- **Encrypt sensitive data at rest** — Use strong encryption standards (AES-256) for stored sensitive data.
- **Minimize data collection** — Only store data you actually need; delete it when no longer required.
- **Avoid exposing sensitive data in URLs** — Query strings can appear in logs and browser history.

---

## 7. Authentication & Session Management

### Best Practices

- **Enforce strong password policies** — Require minimum length, and check against known breached password lists (e.g., via the [Have I Been Pwned API](https://haveibeenpwned.com/API/v3)).
- **Implement multi-factor authentication (MFA)** — Especially for admin accounts and sensitive operations.
- **Use secure, random session tokens** — Generate session IDs using a cryptographically secure random number generator.
- **Invalidate sessions on logout** — Ensure server-side session data is destroyed when a user logs out.
- **Set session expiry** — Expire sessions after a period of inactivity.
- **Regenerate session IDs after login** — Prevent session fixation attacks by issuing a new session token after successful authentication.
- **Rate-limit login attempts** — Implement account lockout or CAPTCHA after repeated failed attempts to prevent brute-force attacks.

---

## 8. General Best Practices

- **Apply the principle of least privilege** — Users and services should have only the minimum permissions necessary.
- **Perform regular security reviews and code audits** — Include security as part of your development process, not just at the end.
- **Use dependency scanning** — Regularly check your dependencies for known vulnerabilities (e.g., `npm audit`, `pip-audit`, Dependabot).
- **Log and monitor** — Maintain audit logs for authentication events, data access, and errors. Set up alerts for suspicious activity.
- **Follow a Secure Development Lifecycle (SDL)** — Incorporate security requirements, threat modeling, and testing at every stage of development.
- **Educate your team** — Security is a shared responsibility. Regular training on secure coding practices is valuable for all developers.

---

## 9. Additional Resources

- [OWASP Top Ten](https://owasp.org/www-project-top-ten/) — The definitive list of the most critical web application security risks.
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — Concise, actionable guidance on specific security topics.
- [OWASP WebGoat](https://owasp.org/www-project-webgoat/) — A deliberately insecure application for safe, hands-on learning.
- [MDN Web Security](https://developer.mozilla.org/en-US/docs/Web/Security) — Browser security documentation and guides.
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework) — A framework for managing and reducing cybersecurity risk.
- [Have I Been Pwned](https://haveibeenpwned.com/) — Check if credentials have been exposed in known data breaches.

---

*This guide is intended for developers building and maintaining web applications. The goal is defensive — understanding these vulnerabilities helps you build more secure software and protect your users.*
