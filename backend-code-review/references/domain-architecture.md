# III. Architecture & Design

## Table of Contents
- [13. Single Responsibility & Layering](#13-single-responsibility--layering)
- [14. Interface-Oriented Programming](#14-interface-oriented-programming)
- [15. Configuration & Code Separation](#15-configuration--code-separation)

---

## 13. Single Responsibility & Layering

### Search Patterns

```
# Controller/Handler directly operating database (cross-layer call signal)
Go:       func Controller / func Handler containing DB, db, sql, Query, Exec, .Find, .Where
Java:     @Controller / @RestController / @GetMapping containing JdbcTemplate, Repository, mapper., session.
Python:   @app.route / @router. / def view containing Model., objects., session., cursor., db.
TypeScript: @Controller / @Get / @Post containing prisma., repository., .query, .raw

# "God function" (function/method over 100 lines)
Go:       awk '/^func /{name=$0; start=NR} /^}$/{if(NR-start>100) print NR, name}' *.go
Java:     awk '/public.*{/{name=$0; start=NR} /^    }$/{if(NR-start>100) print NR, name}' *.java
Python:   awk '/^def /{name=$0; start=NR} /^$/{if(NR-start>100) print NR, name}' *.py
```

### Checklist
- [ ] Are layers clear: Handler/Controller → Service → Repository/DAO
- [ ] Does Controller only do parameter validation and forwarding, no business logic
- [ ] Does Service layer only contain business logic, no SQL/HTTP details
- [ ] Does Repository layer only contain data access, no business judgment
- [ ] Is dependency direction correct (upper layer depends on lower layer, no reverse)
- [ ] Is there cross-layer direct calling

### Deep Dive Method
1. Draw module dependency diagram: who calls whom
2. For functions over 100 lines, analyze if responsibilities can be split
3. Find specific locations of cross-layer calls
4. Give refactored layered code structure

### Scenario Simulation

**Cross-layer call simulation**: If Controller directly operates database, how many places need changing if you switch database implementations later? If you need to add permission check before Service call, how many places to add? If a Service function needs to be reused by both HTTP and gRPC entry points, can it directly handle HTTP request objects?

**God function simulation**: This 100-line function, if it has a bug, can you quickly locate which segment caused it? Can you test one segment independently?

**Dependency direction simulation**: Is there Repository layer depending on Service layer? What problems can circular dependency cause?

> **Deep dive signal**: If cross-layer calls or god functions do exist, require the developer to draw the current call chain diagram, annotate each layer's boundary violation points, and provide a splitting plan.

---

## 14. Interface-Oriented Programming

### Search Patterns

```
# Interface definitions
Go:         type <Name> interface
Java:       interface <Name>
Python:     Protocol, ABC, @abstractmethod
Rust:       trait <Name>
TypeScript: interface <Name>

# Concrete type direct injection (dependency on concrete implementation signal)
Go:         struct embedding *mysql, *redis, *postgres, *Impl
Java:       @Autowired / @Inject / @Resource followed by Impl, MySQL, Redis, Postgres
Python:     from ... import Impl / from ... import mysql / from ... import redis
TypeScript: new Impl / new Repository / new Service
```

### Checklist
- [ ] Does core business logic depend on interfaces/abstractions rather than concrete implementations
- [ ] Is storage layer abstracted as interface (replaceable database implementation)
- [ ] Are external service calls through interfaces (easy to mock for testing)
- [ ] Go: Are interfaces defined by consumers (consumer defines interface)
- [ ] Java: Is programming oriented towards interface rather than Impl
- [ ] Python: Are Protocol/ABC used to define abstractions
- [ ] Rust: Are traits used to define behavior
- [ ] Is interface granularity reasonable (small interfaces优于大接口)

### Deep Dive Method
1. List all struct/class fields/dependency types
2. Mark which are concrete implementations and which are interfaces
3. Analyze if concrete implementation fields should be changed to interfaces
4. Give interface definition and dependency injection examples

### Scenario Simulation

**Replaceability simulation**: If switching MySQL to PostgreSQL, how many files need changing? If writing unit tests for OrderService, can database dependency be mocked? When business layer directly does `new MySQLClient()`, two services share the same business logic but use different storage engines — what then?

**Interface granularity simulation**: How many methods does this interface have? If a caller only uses one method but is forced to depend on all methods, is that reasonable? Is the interface continuously adding methods? Should it be split into smaller interfaces?

> **Deep dive signal**: If core business logic directly depends on concrete implementations, require the developer to indicate how many files need changing to replace the implementation, and provide interface abstraction plan and mock test examples.

---

## 15. Configuration & Code Separation

### Search Patterns

```
# Hard-coded environment configs (ports, addresses) — focus on environment config hard-coding here, general magic values see #5
General: 3306, 5432, 6379, 27017, localhost, 127.0.0.1, 0.0.0.0

# Hard-coded keys/tokens (🔴 security risk)
General: password = "...", secret = "...", token = "...", api_key = "...", apikey = "..."

# Config reading methods
Go:       os.Getenv, viper., config.
Java:     @Value, @ConfigurationProperties, Environment, Config
Python:   os.environ, os.getenv, config., settings., pydantic
Rust:     std::env::, dotenv, config::
TypeScript: process.env, config., ConfigModule
```

### Checklist
- [ ] Are database addresses/ports hard-coded
- [ ] Are keys/tokens hard-coded (security risk!)
- [ ] Are timeout values, retry counts configurable
- [ ] Are different environments (dev/staging/prod) configs separated
- [ ] Do configs have reasonable defaults

### Deep Dive Method
1. Find all hard-coded config values
2. Check if there are corresponding config files or environment variables
3. For keys/tokens: mark as 🔴 CRITICAL security issue
4. Give config management solutions (Go: viper, Java: application.yml, Python: pydantic-settings, Rust: dotenv+config crate)

### Scenario Simulation

**Multi-environment simulation**: Same code, different configs (DB address, key, timeout) between dev and production environments, how to switch? Hard-coding means code needs to be changed and recompiled. What if you forget to change `localhost:6379` when deploying to production?

**Key leak simulation**: Key hard-coded in code, after code enters Git, who can see it? Once key leaks, what's the rotation process? If code is open-sourced or leaked, what can attackers do?

**Config change simulation**: Does modifying a timeout require redeployment? Can it hot-update through config center?

> **Deep dive signal**: If there are hard-coded keys or environment configs, require the developer to explain multi-environment switching mechanism, and provide externalized config transformation plan and key rotation process.

---

## Common Missing Items Reference

| If You See | Must Check For | If Missing, Flag |
|------------|----------------|------------------|
| Controller/Handler | Whether it only does parameter validation + forwarding, no DB operations | ⚠️ Cross-layer call |
| `new MySQLClient()` / `new RedisClient()` | Whether through interface/dependency injection | ⚠️ Dependency on concrete implementation |
| Hard-coded port `3306` / `localhost` | Whether read from config file | ⚠️ Hard-coded config |
| Hard-coded key `password = "..."` | Whether using environment variable/secret management service | ❌ Key leak |
| Service layer directly `db.Query(...)` | Whether through Repository abstraction | ⚠️ Unclear layering |

---

## Cross-Dimension Association Hints

| Association Signal | Points To Dimension | Question to Ask |
|-------------------|---------------------|-----------------|
| Controller directly operates database | #8 Concurrency Safety, #9 Transaction Boundary | Is cross-layer call transaction management standardized? Is concurrency control missed? |
| Key hard-coded | #21 Least Privilege, #20 Input Validation | What account is used? Any root/sa? Is sensitive data also hard-coded? |
| Missing interface | #25 Testing | Core logic depends on concrete implementation, how to mock for unit tests? What happens to test coverage? |
| God function | #18 Retry, #19 Resource Release | Function is too long, is resource release covered in every branch? Is retry logic mixed with business logic? |
| Config not separated | #17 Rate Limiting/Circuit Breaker, #18 Retry | Timeout/retry/rate limit configs hard-coded, are they reasonable across environments? Can they be adjusted as needed? |
