# Engineering Practices Review Checklist

## Checklist

### 9.1 Testing

- [ ] Core business logic has unit tests (payments, amount calculation, state transitions)
- [ ] Tests cover normal / boundary / error scenarios (table-driven tests)
- [ ] Tests are fast (slow tests get skipped; skipped tests don't count)
- [ ] Test code quality equals production code quality
- [ ] No blind pursuit of 100% coverage (core logic must have tests; utilities may vary)

**Test Pyramid:**
```
      ╱  E2E Tests  ╲        ← Few, validate critical paths
     ╱────────────────╲
    ╱  Integration Tests ╲   ← Some, validate module interaction
   ╱────────────────────────╲
  ╱     Unit Tests            ╲  ← Many, cover core logic
```

### 9.2 API Documentation

- [ ] Every endpoint documented: URL, Method, parameters, response, error codes, examples
- [ ] Documentation stays synchronized with implementation (outdated docs are worse than none)
- [ ] Standard formats used: Swagger / OpenAPI
- [ ] Breaking changes have migration plans and transition periods

### 9.3 Change Compatibility

Before every code change, verify:

- [ ] Old version data works with new code
- [ ] Old version clients can still call new endpoints
- [ ] DB schema changes are backward compatible
- [ ] Config changes have defaults; old configs still work
- [ ] Message format changes do not affect in-flight messages
- [ ] Canary deployment is considered when needed

**Safe DB Migration Rules:**
- Adding columns: OK
- Removing columns: phased (stop reading → clean up later)
- Changing types: phased (add new column → migrate data → switch reads → drop old)
- Never directly delete a used column/table; mark deprecated first, clean up next release

### 9.4 Knowledge Management

- [ ] Design docs explain "why" not just "what"
- [ ] Operations docs exist: deployment architecture, runbooks, incident response
- [ ] Changelog maintained per version
- [ ] Technical decisions recorded with ADRs (Architecture Decision Records)
