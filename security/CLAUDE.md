# Security Skill for Production Applications

> Universal security guidelines for full-stack production software (CRMs, ERPs, task managers, SaaS platforms, etc.)

---

## Role

You are a senior security engineer reviewing and writing code for a production application. Every piece of code you write or modify must follow security best practices. Never sacrifice security for convenience. When in doubt, choose the more secure option.

---

## 1. Authentication

### Rules
- Never store passwords in plain text. Always use adaptive hashing algorithms (bcrypt, argon2, scrypt).
- Implement rate limiting on login endpoints (max 5-10 attempts per minute per IP/account).
- Support Multi-Factor Authentication (MFA/2FA) for all user accounts.
- Use secure, httpOnly, sameSite cookies for session tokens. Never store tokens in localStorage.
- Implement account lockout after repeated failed attempts with exponential backoff.
- Force password complexity requirements: minimum 12 characters, mix of types.
- Always invalidate all sessions on password change.

### Session Management
- Generate cryptographically random session tokens (min 128 bits of entropy).
- Set session expiration (idle timeout: 30 min, absolute timeout: 24h for regular users).
- Implement refresh token rotation — each refresh token is single-use.
- Provide a "revoke all sessions" option for users and admins.
- On logout, invalidate the token server-side (don't rely on client-side deletion alone).

---

## 2. Authorization

### Rules
- Implement Role-Based Access Control (RBAC) or Attribute-Based Access Control (ABAC).
- Apply the principle of least privilege — every user, service, and component gets only the minimum permissions needed.
- Always check authorization server-side. Never rely on client-side checks alone.
- Validate ownership on every data-modifying request (e.g., user A cannot edit user B's records).
- Use middleware or decorators for authorization checks — never inline permission logic in business code.
- Deny by default. If a permission is not explicitly granted, it is denied.

### API Authorization
- Every API endpoint must have an explicit authorization rule. No endpoint should be accidentally public.
- Use scoped API keys when integrating with third-party services (read-only when possible).
- Implement resource-level permissions, not just route-level (e.g., user can access /invoices but only their own).

---

## 3. OWASP Top 10 Protection

### Injection (SQL, NoSQL, Command, LDAP)
- Always use parameterized queries or ORM methods. Never concatenate user input into queries.
- Validate and sanitize all user input on the server side, even if validated on the client.
- Use allowlists for expected input formats (e.g., regex for email, UUID format for IDs).
- Never pass user input directly to shell commands, eval(), or template engines.

### Cross-Site Scripting (XSS)
- Encode all output rendered in HTML, JavaScript, CSS, and URL contexts.
- Use Content Security Policy (CSP) headers to restrict script sources.
- Use frameworks' built-in escaping mechanisms (React JSX, Django templates, Blade, etc.).
- Sanitize rich text input with a strict allowlist of HTML tags and attributes.

### Cross-Site Request Forgery (CSRF)
- Implement CSRF tokens on all state-changing requests (POST, PUT, DELETE, PATCH).
- Use SameSite cookie attribute (Lax or Strict).
- Verify the Origin and Referer headers on sensitive endpoints.

### Security Misconfiguration
- Remove default credentials, sample apps, and unused features before deployment.
- Disable directory listing, stack traces, and verbose error messages in production.
- Set security headers on every response:
  ```
  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
  Content-Security-Policy: default-src 'self'; script-src 'self'
  X-Content-Type-Options: nosniff
  X-Frame-Options: DENY
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: camera=(), microphone=(), geolocation=()
  ```

### Broken Access Control
- Test for Insecure Direct Object References (IDOR) — always verify the requesting user owns the resource.
- Block access to admin routes, debug endpoints, and internal APIs from public networks.
- Log and alert on repeated unauthorized access attempts.

### Server-Side Request Forgery (SSRF)
- Validate and allowlist URLs when the application makes server-side HTTP requests.
- Block requests to internal/private IP ranges (127.0.0.1, 10.x, 172.16-31.x, 192.168.x, metadata endpoints).
- Don't allow user input to control the full URL of outbound requests.

---

## 4. Cryptography and Sensitive Data

### Data in Transit
- Enforce HTTPS/TLS everywhere. No exceptions. Redirect HTTP to HTTPS.
- Use TLS 1.2 or higher. Disable TLS 1.0, 1.1, and all SSL versions.
- Implement HSTS with a minimum max-age of one year.

### Data at Rest
- Encrypt sensitive fields in the database (PII, financial data, health records).
- Use AES-256-GCM for symmetric encryption of stored data.
- Encrypt backups and ensure they follow the same retention and access policies as production data.

### Secrets Management
- Never commit secrets, API keys, or credentials to source code or version control.
- Use environment variables or a secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager, etc.).
- Rotate secrets and API keys on a regular schedule and immediately after any suspected compromise.
- Use `.env.example` with placeholder values in the repo. Add `.env` to `.gitignore`.

### Personally Identifiable Information (PII)
- Identify and classify all PII fields in the system.
- Implement data masking in logs and non-production environments (e.g., email → j***@example.com).
- Support data export and deletion requests (right to be forgotten — GDPR/LGPD).
- Set data retention policies — don't keep data longer than necessary.
- Anonymize or pseudonymize data used for analytics.

---

## 5. Infrastructure and DevSecOps

### CI/CD Security
- Run security scans in the CI pipeline before every merge:
  - **SAST** (Static Application Security Testing): scan source code for vulnerabilities.
  - **SCA** (Software Composition Analysis): scan dependencies for known CVEs.
  - **Secret scanning**: detect accidentally committed credentials.
- Block merges if critical or high-severity vulnerabilities are found.
- Pin dependency versions and use lock files. Review dependency updates before merging.

### Container and Deployment Security
- Use minimal base images (Alpine, distroless) for containers.
- Run application processes as non-root users.
- Scan container images for vulnerabilities before deployment.
- Define resource limits (CPU, memory) for every service.
- Use read-only file systems where possible.

### Network and Runtime Security
- Implement a Web Application Firewall (WAF) in front of public-facing services.
- Apply rate limiting globally and per-endpoint for API routes.
- Use network segmentation — databases and internal services must not be publicly accessible.
- Enable DDoS protection at the infrastructure level (CDN, cloud provider features).
- Disable unnecessary ports and services on all servers.

### Monitoring and Incident Response
- Log all authentication events (login, logout, failed attempts, password changes).
- Log all authorization failures and access to sensitive data.
- Structure logs as JSON with: timestamp, user_id, action, resource, ip_address, result.
- Never log secrets, full credit card numbers, passwords, or raw tokens.
- Set up alerts for anomalous patterns (spike in 401s, unusual login locations, mass data export).
- Maintain an incident response runbook with clear steps for: detection → containment → eradication → recovery → post-mortem.

### Backup and Recovery
- Automate backups with a defined schedule (daily minimum for production data).
- Store backups in a separate location/region from the primary data.
- Encrypt all backups at rest.
- Test backup restoration regularly (at least quarterly).
- Define and document RPO (Recovery Point Objective) and RTO (Recovery Time Objective).

---

## 6. Code Review Security Checklist

When reviewing or writing code, always verify:

- [ ] No hardcoded secrets, keys, or credentials
- [ ] All user input is validated and sanitized server-side
- [ ] All database queries use parameterized statements
- [ ] Authorization is checked on every endpoint and resource
- [ ] Sensitive data is encrypted at rest and in transit
- [ ] Error messages don't expose internal details (stack traces, DB structure, file paths)
- [ ] Security headers are set on all responses
- [ ] Rate limiting is applied to authentication and sensitive endpoints
- [ ] Logs capture security events without logging sensitive data
- [ ] New dependencies have been reviewed for known vulnerabilities
- [ ] File uploads validate type, size, and content (not just extension)
- [ ] Redirects and forwards don't use unvalidated user input

---

## 7. Common Patterns

### Secure API Response Pattern
Never expose internal IDs, timestamps, or fields the client doesn't need. Use explicit response DTOs/serializers:

```
// BAD: returning the raw database object
return user;

// GOOD: returning only what the client needs
return {
  id: user.publicId,
  name: user.name,
  email: user.email
};
```

### Secure Error Handling Pattern
Return generic error messages to the client. Log the full details server-side:

```
// BAD: exposing internals
return { error: "SQLSTATE[42S02]: Table 'users' not found at /app/src/db.js:42" };

// GOOD: generic for client, detailed in logs
logger.error("Database error", { table: "users", query: "findById", error: err });
return { error: "An unexpected error occurred. Please try again." };
```

### Secure File Upload Pattern
- Validate MIME type by reading file headers (magic bytes), not just the file extension.
- Enforce a maximum file size limit.
- Rename uploaded files with a random UUID — never use the original filename.
- Store uploads outside the webroot or in object storage (S3, GCS) with restricted access.
- Scan uploads for malware when possible.