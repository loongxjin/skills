# Architecture Design Review Checklist

## Checklist

### 2.1 Layer Boundaries & Responsibilities

- [ ] Controller/Handler layer only does parameter validation, auth check, protocol conversion — no business logic
- [ ] Service/Domain layer only contains business rules, does not directly manipulate HTTP objects (Request/Response)
- [ ] Repository/DAO layer only handles data access, contains no business decisions
- [ ] Dependencies flow strictly top-down; no upward dependencies
- [ ] No Controller calling Repository directly, skipping Service

### 2.2 Interfaces & Dependency Inversion

- [ ] Core logic depends on abstract interfaces, not concrete implementations
- [ ] Interfaces are defined on the consumer side (Go best practice)
- [ ] Dependencies are injected at initialization, not instantiated internally
- [ ] Interfaces are replaceable and mockable for unit testing
- [ ] Adding new implementations does not affect existing logic (Open/Closed Principle)

### 2.3 Config Separation

- [ ] Infrastructure addresses (DB, Redis, MQ) are externalized
- [ ] Timeout and retry parameters are configurable
- [ ] Keys and credentials are managed via Vault/Secret Manager, not hardcoded or in Git
- [ ] Feature flags (canary, degradation) are externalized
- [ ] Business thresholds (limits, rate limits, cache TTL) are configurable
- [ ] Different environments use environment variables or config centers; same code deploys everywhere

### 2.4 Single Responsibility

- [ ] No Service file exceeds 500 lines
- [ ] No class injects a dozen dependencies
- [ ] Changing one business feature does not require modifying the same file in multiple places
- [ ] If the above exist, consider splitting by business domain or technical concern

## Anti-pattern Signals

- **Fat Controller**: Controller contains business logic → not reusable, not testable
- **Protocol Leak**: Service layer depends on HTTP Request/Response → tightly coupled
- **God Class**: One class/file takes on too many responsibilities → high change risk, merge conflicts
- **Hardcoded Config**: Environment-specific values in code → hard to deploy across environments
