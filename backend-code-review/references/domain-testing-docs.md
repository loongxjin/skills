# VI. Testing & Documentation

## Table of Contents
- [25. Critical Path Automated Testing](#25-critical-path-automated-testing)
- [26. Code Review Habits](#26-code-review-habits)
- [27. API Documentation & Code Sync](#27-api-documentation--code-sync)
- [28. Pitfall Self-Check](#28-pitfall-self-check)
- [29. Backward Compatibility](#29-backward-compatibility)

---

## 25. Critical Path Automated Testing

### Search Patterns

```
Test files: Go *_test.go | Java *Test.java *Tests.java *IT.java | Python test_*.py *_test.py | Rust *.rs | TypeScript *.test.ts *.spec.ts
Test functions: Go func Test func Benchmark func Fuzz | Java @Test @ParameterizedTest @RepeatedTest | Python def test_ class Test @pytest | Rust #[test] #[tokio::test] | TypeScript describe( it( test(
Mock: Go gomock testify mockery | Java Mockito @Mock @InjectMocks | Python unittest.mock @patch pytest-mock freezegun | TypeScript jest.fn jest.mock sinon vi.fn vi.mock
Core business: Go func.*Pay func.*Charge func.*Deduct func.*Transfer func.*Calculate | Java public.*pay public.*charge public.*deduct public.*transfer public.*calculate | Python def.*pay def.*charge def.*deduct def.*transfer def.*calculate
```

### Checklist
- [ ] Is core business logic (payment, status flow, amount calculation) covered by unit tests
- [ ] Are table-driven tests/parameterized tests used to cover boundary cases
- [ ] Are there integration tests covering critical chains
- [ ] Are external dependencies mocked
- [ ] Do tests have meaningful assertions
- [ ] Are exception paths covered

### Deep Dive Method
1. List all core business functions
2. Compare with test files, find core functions without test coverage
3. Analyze assertion quality of existing tests (real assertion or `assert True`)
4. **Focus on boundary tests**: 0, negative numbers, null, max values, concurrent scenarios
5. Give missing test cases:
   - Go: table-driven test `[]struct{ name string; input ...; want ... }`
   - Java: `@ParameterizedTest` + `@MethodSource`
   - Python: `@pytest.mark.parametrize`
   - TypeScript: `test.each()`

### Scenario Simulation

**No test simulation**: Payment amount calculation logic changed one line, no unit tests. After going live, found that 100 yuan orders charged 1000 yuan. How long until discovered? How many user complaints needed?

**Test quality simulation**: Tests only verified happy path, were boundary conditions tested? Amount of 0, negative, super large? Concurrent scenarios? Network timeout?

**Mock vs real simulation**: Mocked database returning success, but real database has unique constraints. Can unit tests find this bug? Need integration tests?

**State machine simulation**: Payment logic has 4 states (pending, paid, refunded, cancelled) and 6 state transition rules. Can manual testing cover all combinations? If adding 5th state "partial refund", can you ensure existing logic isn't broken? Without unit tests, would you dare to change?

> **Deep dive signal**: If core business logic has no test coverage, or tests only have happy path, immediately ask about boundary condition and exception path test coverage.

---

## 26. Code Review Habits

This dimension is process review, not targeting specific code. Focus on checking:

### Checklist
- [ ] Are changes split into reasonably sized PRs (not exceeding 400 lines)
- [ ] Does PR description clearly explain "why" the change
- [ ] Are there corresponding test changes
- [ ] Are there breaking changes that need to notify downstream

### Scenario Simulation
> This PR has 2000 lines of changes across 30 files. Can the reviewer carefully read through it? Or just skim and Approve? If there's a bug hidden inside, can it be found?

---

## 27. API Documentation & Code Sync

### Search Patterns

```
Documentation annotations: Go @Router @Summary @Tags @Param @Success @Failure rpc message service(.proto) | Java @ApiOperation @Api @Schema @Operation @Tag @ApiResponse | Python swagger openapi @swagger serializers. | TypeScript @ApiOperation @ApiResponse @ApiTags swagger openapi trpc.
Route definitions: Go router. r.GET r.POST r.PUT r.DELETE Group Handle | Java @GetMapping @PostMapping @PutMapping @DeleteMapping @RequestMapping | Python @app.route @router. path( | TypeScript @Get @Post @Put @Delete router.
```

### Checklist
- [ ] Does each API have documentation annotations
- [ ] Do parameters and return values in documentation match code
- [ ] Are there deprecated APIs not marked in documentation
- [ ] Does complex business logic have design documents

### Deep Dive Method
1. List all route definitions in code
2. Compare with interfaces defined in documentation
3. Mark: code has but documentation doesn't, documentation has but code doesn't, parameter inconsistencies
4. Give documentation update suggestions

### Scenario Simulation

**Documentation inconsistency simulation**: Documentation says returns `user_id`, code returns `userId`. Frontend developed according to documentation, after going live parsing fails. More dangerous: documentation says `status: number`, code already changed to `status: string`, downstream parses as number, all errors in production. Inconsistent documentation and code is more dangerous than no documentation.

**Deprecated API simulation**: An interface has been marked deprecated for 3 months, are there still callers using it? What if directly deleted? Is there a decommission plan?

**Field change simulation**: Added a required field without updating documentation, caller doesn't know to pass this field, production environment returns 400.

> **Deep dive signal**: Documentation and code inconsistency is the source of collaboration problems. Discovering any field name, type, required/optional inconsistency should be immediately flagged and impact scope tracked.

---

## 28. Pitfall Self-Check

Actively think about the following questions for each function during review:

### Checklist
- [ ] **Null pointer/None/null**: Can this variable be nil/null/None?
- [ ] **Out of bounds**: Can slice/array/list/map access go out of bounds? Does key exist?
- [ ] **Division by zero**: Is there division operation? Can divisor be 0?
- [ ] **Concurrency**: What happens if multiple threads/goroutines access this code simultaneously?
- [ ] **Boundary**: Empty input? Super long input? Special characters?
- [ ] **Timezone**: Is timezone considered in time handling?
- [ ] **Encoding**: Is encoding considered in string handling?
- [ ] **Precision**: Is decimal used instead of float for amount calculation?
- [ ] **Order**: Does this logic depend on specific execution order?

### Deep Dive Method
1. For each function, answer the above questions one by one
2. For places that may error, give specific protective code
3. Pay special attention to high-risk logic like amounts, state flows, permission judgments

### Scenario Simulation

**Null pointer simulation**: Does this function's return value have null check at caller? If database query returns nil/null, will downstream code panic/NPE?

**Amount precision simulation**: `0.1 + 0.2 == 0.3` is `False` in float. Are amount fields using float or decimal? `price * quantity` error amplifies when unit price is small and quantity is large. What's the accumulated error for 10000 times `0.1 + 0.2`?

**Timezone simulation**: `time.Now()` returns server local time. If server is in Shanghai (UTC+8) and database stores in UTC, query `WHERE created_at > '2024-01-01'` will be off by 8 hours. What about cross-timezone queries?

**Division by zero simulation**: Can average value divisor be 0? Can percentage calculation denominator be 0?

> **Deep dive signal**: Pitfalls are high-frequency bug areas. Null pointers, amount precision, timezone, division by zero — every seemingly small issue can cause large-scale failures in production. Reviewers should check one by one, do not skip.

---

## 29. Backward Compatibility

### Search Patterns

```
Schema changes: ALTER TABLE ADD COLUMN DROP COLUMN MODIFY COLUMN CREATE TABLE
Serialization tags: Go json: xml: form: query: gorm: | Java @JsonProperty @XmlElement @Column | Python Field( serializer @dataclass pydantic | TypeScript @ApiProperty @IsOptional interface
Config items: Config config .Env viper pydantic
Enums/Constants: const iota enum Enum Enumeration
```

### Checklist
- [ ] Do new fields have default values (old data doesn't have this field)
- [ ] Does deleting/renaming fields affect old version clients
- [ ] Does enum value addition affect old logic
- [ ] Do new config items have default values
- [ ] Are database schema changes backward compatible
- [ ] Is API version management strategy clear

### Deep Dive Method
1. Find struct/class/DB/API changes involved in this change
2. **Simulate old version client calling new interface**: Which fields will be missing? What happens after missing?
3. **Simulate old data processed by new code**: New field is NULL for old data, can code handle correctly?
4. Give compatibility fixes

### Scenario Simulation

**Old client simulation**: New API deleted `discount_price` field, old client code `if (response.discount_price > 0)` becomes `undefined > 0`, which is `false` in JavaScript — business logic error but no error reported. Silent errors are the scariest. What will the 10% of old clients reading this field see? Crash? Blank?

**Database change simulation**: Added a NOT NULL field to table without default value, old code inserting data will error. Is deployment order code first or database first? If DB adds `is_verified` field without default value, old data is all NULL, `if user.is_verified { ... }` doesn't apply to old users — what if it affects risk control logic?

**Enum addition simulation**: Added a value to enum, can old version switch/match handle this new value? Will it go to default branch? Is default branch behavior correct?

> **Deep dive signal**: Backward compatibility issues often don't expose during development, but concentrate during deployment or version switching. Any schema, enum, or API field change should simulate new-old version coexistence scenarios.

---

## Common Missing Items Reference

| If You See | Must Check For | If Missing, Flag |
|------------|----------------|------------------|
| Core business function (Pay/Charge/Calculate) | Unit tests | ⚠️ No test coverage |
| Test file | Boundary tests (0/null/negative/max) | ⚠️ Insufficient tests |
| API route definition | Corresponding documentation annotations | ⚠️ Missing documentation |
| `ALTER TABLE DROP COLUMN` | Is there gray-scale strategy (code compatibility first, then delete field) | ❌ Breaking change |
| New struct field | Is there default value / `omitempty` | ⚠️ Old data compatibility |
| Enum new value | Is old logic compatible with new value | ⚠️ Enum extension |
| `json:"field_name"` | Does field name change affect old clients | ❌ Field rename |
| Amount calculation | Is decimal used (not float) | ❌ Precision loss |
| `time.Now()` | Is timezone considered | ⚠️ Timezone issue |
