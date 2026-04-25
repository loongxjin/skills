# I. Code Fundamentals

## Table of Contents
- [1. Naming Conventions](#1-naming-conventions)
- [2. Single Function Responsibility](#2-single-function-responsibility)
- [3. Parameter Minimization](#3-parameter-minimization)
- [4. Control Flow Flattening](#4-control-flow-flattening)
- [5. Eliminate Magic Values](#5-eliminate-magic-values)
- [6. Code Reuse & Deduplication](#6-code-reuse--deduplication)

---

## 1. Naming Conventions

### Search Patterns

Search keywords: tmp, temp, ret, data, info, result, obj, val, buf, num
Applicable files: *.go, *.java, *.py, *.rs, *.ts, *.tsx

### Checklist
- [ ] Do variable names express business meaning (`orderCount` instead of `cnt`)
- [ ] Are function names verb phrases (`GetUserByID` / `getUserById` / `get_user_by_id`)
- [ ] Do bool variables start with `is/has/can/should`
- [ ] Are constants named per language convention (Go: PascalCase, Java: UPPER_SNAKE, Python: UPPER_SNAKE)
- [ ] Do file names reflect content (`user_service.go` / `UserService.java` / `user_service.py` / `user_service.rs`)
- [ ] Are there numbered names like `data1, data2, result1, result2`

### Deep Dive Method
1. Find each suspicious name
2. Trace the variable's usage chain in the function, confirm if meaning can be understood from context
3. Give specific renaming suggestions

### Scenario Simulation

For each vague name (e.g., `data`, `result`, `info`), ask:

- **Will you still understand this in six months?** If a variable is called `data`, you need to scroll back and forth in the function to understand what it is, it should be renamed
- **Will it be misunderstood during code review?** Is `result` a success result or error result? Is `info` user info or order info?
- **Can grep pinpoint it?** Searching `data` matches the entire screen; searching `orderConfirmData` only matches where you want
- **Can a new teammate understand it?** Seeing `result2`, can they know what it represents without reading context?

> **Deep dive signal**: If a function has multiple `data`/`result`/`info` variables, it usually means the function does too much. Combine with #2 Single Function Responsibility for review.

---

## 2. Single Function Responsibility

### Search Patterns

Language-specific function definitions: Go `func`, Java `public/private/protected`, Python `def`, Rust `fn`, TypeScript `function`/`=>`/`async`
Multi-responsibility signal: function name contains `and`/`or` (e.g., `CreateAndSend`, `parse_or_default`)
Applicable files: *.go, *.java, *.py, *.rs, *.ts, *.tsx

### Checklist
- [ ] Does function exceed 50 lines (alert if so)
- [ ] Does function name imply multiple actions (`CreateAndSendEmail` / `create_and_send_email`)
- [ ] Are operations of different abstraction levels mixed in the function
- [ ] Is there a code segment that can be extracted as an independent and reusable function

### Deep Dive Method
1. For functions over 50 lines, annotate what each segment does
2. If a segment exceeds 10 lines and is logically independent from surrounding code, suggest extraction
3. Give example function signatures after splitting

### Scenario Simulation

For each overly long function, ask:

- **Will changing one responsibility accidentally break another?** E.g., in `CreateAndSendOrder`, changing send logic accidentally affects creation logic
- **Can each behavior be tested independently?** If not, coupling is too tight
- **Does the caller really need both things at once?** Some scenarios only need creation without sending
- **Can you quickly locate issues when it errors?** If the function errors, can you quickly determine which responsibility caused it from the function name?

> **Deep dive signal**: "Paragraph comments" in functions (`// Step 1: create order`, `// Step 2: send notification`) indicate it's doing two independent things. If paragraphs share state (temporary variables), pay special attention to data transfer when splitting. Combine with #13 Layering to check if operations of different abstraction levels are mixed.

---

## 3. Parameter Minimization

### Search Patterns

Multi-parameter signal: 4 or more parameters in function signature (regex match `..., ..., ..., ...`)
Bool parameter: `bool` (Go/Python), `boolean`/`Boolean` (Java) appearing in parameter list
Applicable files: *.go, *.java, *.py, *.rs, *.ts

### Checklist
- [ ] Are there more than 3 parameters
- [ ] Are there bool parameters controlling if-else branches (should be split into two functions)
- [ ] Do the same group of parameters appear repeatedly across multiple functions (should be encapsulated as struct/DTO/dataclass)
- [ ] Is parameter order consistent with language convention (Go: ctx first, Java: request first, Python: self first)

### Deep Dive Method
1. For functions with more than 3 parameters, analyze which parameters naturally belong together
2. Give encapsulation suggestions: Go uses struct, Java uses DTO/RequestObject, Python uses dataclass, Rust uses struct
3. For bool parameters, give two function signatures after splitting

### Scenario Simulation

- **Can the compiler catch parameter swap errors?** `SendEmail(user, subject, body, true, false)` — swapping the 4th and 5th bool parameters, compiler won't complain, but behavior is completely different
- **Do all callers need to change when adding a parameter?** More parameters mean larger change surface when adding new ones
- **Are error handling paths the same for both bool branches?** If different, they should be two functions

> **Deep dive signal**: If the same group of parameters repeatedly appears in multiple functions (e.g., `userID, orderID, amount`), this is not just a parameter minimization issue — it may indicate a missing domain object (e.g., `OrderContext`). Combine with #13 Layering and #14 Interface-Oriented Programming to check if necessary abstractions are missing.

---

## 4. Control Flow Flattening

### Search Patterns

Deep nesting: 3 or more indentation levels (tab or 8+ spaces)
Else branch: `} else {` (Go/Java), `else:` (Python)
Applicable files: *.go, *.java, *.py, *.rs, *.ts, *.tsx

### Checklist
- [ ] Is there if-else nesting deeper than 2 levels
- [ ] Can early return/break/continue reduce nesting
- [ ] Is error handling before normal logic (guard clause pattern)
- [ ] Are there complex conditional expressions that can be extracted as meaningful bool variables

### Deep Dive Method
1. For each deep nesting, draw the control flow structure
2. Show how to rewrite with guard clause
3. Compare readability before and after rewrite

**Go Example:**
```go
// Bad
if err == nil {
    if user != nil {
        // happy path
    }
}

// Good
if err != nil { return err }
if user == nil { return nil }
// happy path
```

**Java Example:**
```java
// Bad
if (result != null) {
    if (result.isValid()) {
        // happy path
    }
}

// Good
if (result == null) { return; }
if (!result.isValid()) { return; }
// happy path
```

**Python Example:**
```python
# Bad
if data is not None:
    if data.is_valid():
        # happy path

# Good
if data is None:
    return
if not data.is_valid():
    return
# happy path
```

### Scenario Simulation

For each deep nesting, ask:

- **Is every branch's error path handled?** The deeper the nesting, the easier to miss error handling for some branch
- **Is the happy path hidden in the innermost layer?** If you need to scroll to the deepest level to see normal logic, nesting is problematic
- **Which branch is most likely to have bugs?** Else branches in nested code are least tested and most likely to hide bugs
- **What if you need to add a condition?** In 4-layer if, which layer do you add to? What happens to readability after adding?

> **Deep dive signal**: Deep nesting often accompanies implicit business rules. `if A then if B then if C` is actually a business condition `A && B && C`, but split into three layers. This not only affects readability but also means if B fails, the developer may forget to handle it. Combine with #28 Pitfall Self-Check to verify if every branch's error path is handled.

---

## 5. Eliminate Magic Values

### Search Patterns

Hard-coded numbers: bare numbers other than 0/1/-1 in conditions or assignments
Hard-coded strings: `"..."` assignment without `final`/`static final`/enum modifier
Applicable files: *.go, *.java, *.py, *.rs, *.ts

### Checklist
- [ ] Do numeric constants have names (`maxRetryCount` instead of `3`)
- [ ] Are states/types defined with enum/constant groups
- [ ] Do error codes have constant definitions
- [ ] Are timeout values, retry counts, etc. configurable
- [ ] Go: uses `const` or `iota`; Java: uses `enum`; Python: uses `Enum` class

### Deep Dive Method
1. List all hard-coded values
2. Analyze which are true magic numbers and which are mathematically fixed (e.g., `*2`, `/2`)
3. Give corresponding language constant/enum definition code

### Scenario Simulation

For each magic value, ask:

- **If business rules change, how many places need updating?** Status code `3` appears in 5 files, business says "cancelled" should change to `4`, can you ensure all are updated?
- **Can a newcomer understand this number's meaning?** `if status == 3` vs `if status == OrderStatus.Cancelled`
- **Is this timeout reasonable across all environments?** 3 seconds is enough for local testing, production may need 10 seconds. Hard-coding means it can't be adjusted per environment
- **Same number, different meaning?** Another developer also used `3` but represents a different business meaning, what happens when searching?

> **Deep dive signal**: If hard-coded numbers are used for status judgment, check if status values are missed. E.g., code only handles `status == 1` and `status == 2`, but DB actually has 3 statuses, the 3rd status gets wrong default handling. Combine with #10 Data Growth Sensitivity and #28 Pitfall Self-Check for "boundary" checks.

---

## 6. Code Reuse & Deduplication

> **Core Focus**: Eliminate copy-paste style duplicate code, extract general logic and patterns, reduce maintenance cost (one modification affects multiple places).

### Search Patterns

Duplicate error handling: `if err != nil` (Go), `catch (` (Java), `except ` (Python) appearing frequently in the same file
Duplicate validation logic: required, @NotNull, @NotBlank, validate, etc. keywords
Duplicate CRUD templates: Create/Update/Delete function signatures are highly similar
Duplicate utility functions: Is/Has/Check/Parse/Format prefixed functions with same implementation in multiple places
Duplicate config constants: const, static final, MAX_, MIN_, DEFAULT_ defined in multiple places
Applicable files: *.go, *.java, *.py, *.rs, *.ts

### Checklist

#### Code Block Duplication
- [ ] Are there copy-pasted code blocks (3+ lines identical or highly similar)
- [ ] Do multiple functions have the same error handling pattern (should extract middleware or common wrapper)
- [ ] Do multiple interfaces have the same parameter validation logic (should extract common validation function)

#### Business Logic Duplication
- [ ] Is the same business rule implemented in multiple places (e.g., order status judgment, permission check)
- [ ] Do similar branch conditions appear in multiple functions (should unify as strategy/state machine)
- [ ] Is there massive duplicate field mapping in data transformation logic (DTO ↔ Model)

#### Template Code Redundancy
- [ ] Are CRUD operations written with the same template code for every entity (consider generic base class/code generation)
- [ ] Does Repository/DAO layer have many similar CRUD methods
- [ ] Can generics/template method pattern reduce duplication

#### Config & Definition Duplication
- [ ] Are the same constants/configs defined in multiple files
- [ ] Are the same error codes/status codes scattered in multiple places
- [ ] Can the same struct/class definitions be merged

### Deep Dive Method

1. **Locate duplicates**: Group by file and count, high-frequency patterns are duplication signals

2. **Analyze reuse value** (not all duplication needs elimination):
   - Two identical places with high business relevance → Must extract
   - Two similar places but different contexts → Merge and abstract first
   - Two identical places but likely to evolve independently → Can temporarily keep

3. **Give refactoring suggestions**:
   - Duplicate code blocks → Extract as common function, parameterize differences
   - Duplicate error handling → Extract middleware or `WrapHandler` wrapper
   - Duplicate validation → Extract common validation function or validation tags
   - Duplicate business logic → Unify to Service layer or domain model
   - Duplicate CRUD → Use generic base class (Go: generics + interface; Java: generic Repository; Python: abstract base class)
   - Duplicate constants → Centralize in constants package/class

### Scenario Simulation

For each duplicate code group, ask:

- **What happens if you modify one side but forget the other?** This is the biggest risk of duplicate code. E.g., payment validation logic exists in two Services, later added coupon validation but only changed one place
- **Are there subtle differences in the two "identical" pieces?** Code that looks the same may have subtle differences in boundary condition handling, error code return. Determine if differences are intentional (preserve when merging) or accidental (which version to use when merging?)
- **Does duplicate business rules indicate a missing domain model?** If "order status flow" logic is scattered in 5 places, it may not be a code reuse problem but a missing `OrderStateMachine` domain object

> **Deep dive signal**: The most dangerous thing about duplicate code is not maintenance cost but **consistency risk**. When one copy of duplicate business logic is modified but the other isn't, a hidden bug is created. Combine with #16 Idempotency Design to check if duplicate write operation logic is consistent, and with #28 Pitfall Self-Check to verify if duplicate boundary condition handling is consistent.

---

## Common Missing Items Reference

| If You See | Must Check For | If Missing, Flag |
|------------|----------------|------------------|
| Function name contains `and`/`or` | Whether it can be split into single-responsibility functions | ⚠️ Multi-responsibility function |
| 3+ parameters | Whether encapsulated as struct/DTO | ⚠️ Too many parameters |
| `true`/`false` as parameter | Whether split into two semantically clear functions | ⚠️ Boolean trap |
| 3+ levels nesting | Whether guard clause flattening is possible | ⚠️ Arrow code |
| Bare numbers `== 3` / `>= 200` | Whether extracted as named constants/enums | ⚠️ Magic values |
| Similar code blocks appearing 3+ times | Whether extracted as common function | ⚠️ Duplicate code |

---

## Cross-Dimension Association Hints

After reviewing code fundamentals dimensions, check the following association risks:

| Association Signal | Points To Dimension | Question to Ask |
|-------------------|---------------------|-----------------|
| Magic values used for status judgment | #8 Concurrency Safety, #16 Idempotency | Is state transition atomic? Will status value changes cause incomplete logic branch coverage? |
| Function responsibility mixes business logic and external calls | #9 Transaction Boundary, #17 Rate Limiting/Circuit Breaker | After splitting, is external call outside transaction? Are there timeout and circuit breaker? |
| Duplicate business rules scattered in multiple places | #25 Testing, #16 Idempotency | Can all copies be kept consistent when modifying? Are all copies covered by tests? |
| Deep nesting contains error handling | #12 Logging, #19 Resource Release | Are inner branch errors correctly handled and logged? Are resources released in all branches? |
| Bool parameter controls different business branches | #16 Idempotency | Are idempotency strategies consistent for both branches? What if caller passes wrong parameter? |
| Hard-coded timeout/retry config | #18 Retry Strategy | Can timeout values be adjusted per environment? Are retry counts reasonable? |
