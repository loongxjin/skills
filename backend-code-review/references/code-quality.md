# Code Quality Review Checklist

## Checklist

### 1.1 Naming

- [ ] Variables clearly convey intent; avoid `d`, `data`, `tmp`
- [ ] Functions start with verbs describing concrete actions (`validateUserPermission` not `process`)
- [ ] Class names are nouns reflecting role responsibility (`PaymentProcessor` not `Manager`)
- [ ] Constants replace magic values (`MAX_RETRY_COUNT` not `60`)
- [ ] File names reflect module responsibility (`payment_processor.go` not `util.go`)
- [ ] Boolean variables use `is/has/can/should` prefix
- [ ] Naming length matches scope (global names are long and explicit)
- [ ] Same name is not reused with different meanings in different contexts

### 1.2 Function Design

- [ ] Function length is under 20 lines (complex logic under 50 at most)
- [ ] Parameter count is ≤ 3; more should be wrapped in a struct
- [ ] Function does one thing only
- [ ] Function does not mix different levels of abstraction
- [ ] Side effects are reflected in function name and comments
- [ ] Uses guard clauses for early returns; avoids arrow-shaped nesting

### 1.3 Magic Values

- [ ] No unexplained literal numbers or strings in code
- [ ] Business status codes use enums/constants
- [ ] Configuration thresholds are extracted as named constants or config items
- [ ] Timeout and retry counts are structured configuration
- [ ] Prefer method encapsulation (`user.IsActive()` over `user.Status == 3`)

### 1.4 Code Duplication

- [ ] No code copied three or more times (rule of three)
- [ ] Extraction preserves semantic independence — code in different business contexts should not be forced together
- [ ] Prefer composition over inheritance for code reuse

### 1.5 Logging

- [ ] Critical business paths have logging
- [ ] Logs contain enough context for troubleshooting (IDs, status, key data, error cause)
- [ ] Log levels are correct (DEBUG / INFO / WARN / ERROR)
- [ ] No INFO or higher level logging inside loops
- [ ] Sensitive information (passwords, tokens, IDs, phone numbers) is masked
- [ ] Structured logging is used for searchability
