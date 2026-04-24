# Security Defense Review Checklist

## Checklist

### 7.1 Input Validation

- [ ] Controller layer validates parameter types (required, format, range)
- [ ] Validator framework used for declarative validation
- [ ] Service layer validates business rules (state, permissions, cross-field checks)
- [ ] Database layer has unique constraints, foreign keys, CHECK constraints as final guard

### 7.2 Injection & Attack Prevention

- [ ] All SQL uses parameterized queries / prepared statements (string concatenation prohibited)
- [ ] XSS prevented (output encoding for HTML/CSS/JS contexts)
- [ ] CSRF prevented (SameSite Cookie + CSRF Token)
- [ ] Replay attacks prevented (request signature + timestamp + nonce)

### 7.3 Authorization & Least Privilege

- [ ] Endpoints check login state
- [ ] Data-level authorization exists (not just login check; verify data ownership)
- [ ] Least privilege principle applied:
  - DB account has only DML permissions, no DDL
  - API tokens have minimal required scope
  - Application process does not run as root
  - Service-to-service calls use internal network + mTLS

### 7.4 Sensitive Data Protection

**Storage:**
- [ ] Passwords hashed with bcrypt/scrypt/argon2 (no plaintext)
- [ ] PII (ID numbers, card numbers) encrypted at rest (AES-256-GCM)
- [ ] Backup files are encrypted

**Transmission:**
- [ ] External communication uses TLS 1.2+
- [ ] Internal service calls use TLS or mTLS
- [ ] API keys/tokens passed via Header, not URL parameters

**Display:**
- [ ] Sensitive data masked in logs
- [ ] Sensitive fields masked in API responses
- [ ] Error messages do not expose internal details (stack traces, SQL, file paths)

**Key Management:**
- [ ] Keys are never hardcoded in code
- [ ] Keys are never committed to Git
- [ ] KMS/Vault used for centralized key management
- [ ] Different environments use different keys
- [ ] Keys support periodic rotation

**Masking Rules Reference:**
```
Phone:     138****1234        (keep first 3 and last 4)
ID:        310***********1234 (keep first 3 and last 4)
Card:      **** **** **** 5678 (keep last 4)
Email:     c***@example.com   (keep first letter and domain)
Password:  [REDACTED]         (fully hidden)
Token:     tk_****...****89ab (keep prefix and last 4 for debugging)
```
