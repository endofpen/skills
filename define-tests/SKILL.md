---
name: define-tests
description: Workflow step "define tests" – after implementation on the current branch, create test files (Kotlin, JUnit 5) for new and functionally changed classes. Produces only empty test methods with backtick names, // arrange // act // assert and TODO(), no test logic. Use for "define tests", "Tests definieren", "Test definieren", "specify test cases".
---

# Define tests

This step comes **after implementation**. The production code on the current branch is already written. Your job is to decide **which tests must exist** for the new and functionally changed classes, and to create them as empty test methods.

You do **not** write test logic, create mocks, or change production code. Implementing the tests is a later step.

## Workflow

### 1. Find changed classes

1. Determine the base branch. Check `origin/main`, `origin/master`, `origin/develop` in that order. If several exist and it is unclear which one the branch was created from, ask the user.
2. List the changed Kotlin files in production code:
   ```bash
   git diff --name-status $(git merge-base HEAD <base>)...HEAD -- '*.kt' ':!**/src/test/**'
   ```
   Also include uncommitted changes (`git status`).
3. Classify:
   - `A` (added) → the class gets tests.
   - `M` / `R` (modified, renamed) → inspect the diff (`git diff <merge-base>...HEAD -- <file>`). Only test on a **functional** change. Pure renames, formatting, imports, KDoc, logging, or refactorings without behavior change do not count.
   - `D` (deleted) → ignore.
4. Skip (but list in the final report):
   - pure data holders without logic (`data class` with properties only, DTOs, enums without methods)
   - interfaces without default implementations
   - pure configuration (e.g. `@Configuration` classes containing only bean definitions), constants, generated code

### 2. Determine the methods to test

Only **public** methods are tested.

- In Kotlin, **everything without a modifier is public**. `private`, `protected`, and `internal` are not tested.
- Do not test: methods generated for a `data class` (`equals`, `hashCode`, `toString`, `copy`, `componentN`), plain property getters/setters.
- Public functions in a `companion object` and public top-level/extension functions are included.
- For **new** classes: all public methods.
- For **modified** classes: new public methods and public methods whose behavior changed. This includes changes in a private helper that is called by the public method.

### 3. Unit test or integration test?

The default is a **unit test per class**.

Define an **integration test across several classes** only when the class is hard to test in isolation, for example:
- collaborators are created internally instead of injected (`val x = Foo()` in the body, stateful `object` singletons)
- the behavior only emerges from several classes working together, and mocks would just replicate the implementation 1:1
- the class is a thin orchestration layer whose only purpose is chaining other classes correctly

Justify every decision for an integration test in the final report.

For integration tests: first search the repo for existing integration tests (e.g. files ending in `IT.kt` or `IntegrationTest.kt`, separate source sets like `src/integrationTest`) and follow their location and naming suffix. If there are none, create `<ClassName>IntegrationTest.kt` next to the unit tests and point this out in the report.

### 4. Derive test cases

Derive test cases **from the actual code**. For each public method, check:

- **Happy path**: the regular expected result
- **Branches**: every functionally relevant branch of `if` / `when` / the Elvis operator `?:`
- **Error cases**: every `throw`, `require`, `check`, `error()`, `requireNotNull`
- **Boundary values**: empty collections, `0`, negative values, min/max, condition boundaries (`>` vs. `>=`)
- **Nullable parameters**: behavior on `null`
- **Side effects**: if the purpose of the method is a call to a collaborator (save, send, publish), a test that this call happens, or does not happen in the error case

Rules:
- **One behavior per test.** No catch-all tests.
- Do not invent behavior the code does not have. If something looks like a bug, do not define a test that locks the bug in; mention it in the report instead.
- No tests for trivial delegation without its own logic, unless the delegation is the functional purpose.

### 5. Write the test files

**Location:** same module, same package, `src/main/kotlin/...` → `src/test/kotlin/...`, file name `<ClassName>Test.kt`.

**If the test file already exists:** do not modify, delete, or reorder existing tests. Insert new test methods into the matching method section. If the section is missing, add it at the end. No duplicates of tests that already exist.

**Structure:**
- `package` same as the production class
- only import: `org.junit.jupiter.api.Test`
- **no** mocks, no system under test, no `@BeforeEach`, no fields, no assertions
- one section per tested method, introduced by `// --- <methodName> ---`, in the order the methods appear in the production class

**Test method – exactly this format:**

```kotlin
    @Test
    fun `<methodName> should <expected result> when <condition>`() {
        // arrange
        // act
        // assert
        TODO()
    }
```

**Naming:**
- English, pattern `` `<methodName> should <expected result> when <condition>` ``
- For the happy path without a special condition, `when …` may be omitted.
- Be specific: name exception types and status values (`should throw CustomerNotFoundException`, `should set status to CANCELLED`).
- For overloaded methods, make the variant distinguishable via the condition.
- Do **not** use these characters in backtick names (invalid on the JVM): `.` `;` `[` `]` `/` `<` `>` `:` `\`

### 6. Verify

1. Detect the build tool (`gradlew` or `mvnw`) and **only compile the tests**, e.g. `./gradlew compileTestKotlin` or `./mvnw test-compile`. In multi-module projects, only the affected modules.
2. Do **not** run the tests: because of `TODO()` they fail on purpose (`NotImplementedError`) until they are implemented.
3. Fix any compilation errors.

### 7. Final report

At the end, briefly output:
- per class: test file (new or extended) and number of defined tests
- integration tests with justification
- skipped classes and methods with reason
- observations in the production code (possible bugs, hard-to-test spots)

## Example

Production class:

```kotlin
class OrderService(
    private val customerRepository: CustomerRepository,
    private val orderRepository: OrderRepository,
) {
    fun createOrder(customerId: Long, items: List<Item>): Order {
        require(items.isNotEmpty()) { "items must not be empty" }
        val customer = customerRepository.findById(customerId)
            ?: throw CustomerNotFoundException(customerId)
        return orderRepository.save(Order(customer, items, OrderStatus.CREATED))
    }

    fun cancelOrder(orderId: Long) {
        val order = orderRepository.getById(orderId)
        check(order.status != OrderStatus.SHIPPED) { "shipped orders cannot be cancelled" }
        orderRepository.save(order.copy(status = OrderStatus.CANCELLED))
    }

    private fun audit(order: Order) { /* ... */ }
}
```

Expected test file `src/test/kotlin/com/example/order/OrderServiceTest.kt`:

```kotlin
package com.example.order

import org.junit.jupiter.api.Test

class OrderServiceTest {

    // --- createOrder ---

    @Test
    fun `createOrder should save order with status CREATED when customer exists`() {
        // arrange
        // act
        // assert
        TODO()
    }

    @Test
    fun `createOrder should throw CustomerNotFoundException when customer does not exist`() {
        // arrange
        // act
        // assert
        TODO()
    }

    @Test
    fun `createOrder should throw IllegalArgumentException when items are empty`() {
        // arrange
        // act
        // assert
        TODO()
    }

    @Test
    fun `createOrder should not save order when customer does not exist`() {
        // arrange
        // act
        // assert
        TODO()
    }

    // --- cancelOrder ---

    @Test
    fun `cancelOrder should save order with status CANCELLED when order is not shipped`() {
        // arrange
        // act
        // assert
        TODO()
    }

    @Test
    fun `cancelOrder should throw IllegalStateException when order is already shipped`() {
        // arrange
        // act
        // assert
        TODO()
    }
}
```

`audit` is `private` and therefore gets no test of its own.