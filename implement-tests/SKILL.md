---
name: implement-tests
description: Workflow step "implement tests" – implement the empty test methods created by the define-tests step (bodies with // arrange // act // assert and TODO()) in Kotlin using JUnit 5, MockK and Kotest assertions, then run them until green. Use for "implement tests", "Tests implementieren", "Tests ausimplementieren", "fill in tests".
---

# Implement tests

This step comes **after "define tests"**. The current branch contains test files with test methods that only have a name, the comments `// arrange`, `// act`, `// assert`, and `TODO()`. Your job is to implement exactly these tests.

You do **not** change production code, rename tests, delete tests, or add new tests. The test names are the specification.

## Workflow

### 1. Find the tests to implement

1. If the user names specific test files or classes, use those.
2. Otherwise determine the base branch (`origin/main`, `origin/master`, `origin/develop`, in that order; ask if unclear) and collect test files changed on the branch, including uncommitted and untracked files:
   ```bash
   git diff --name-only $(git merge-base HEAD <base>)...HEAD -- '**/src/test/**/*.kt'
   git status --porcelain -- '**/src/test/**/*.kt'
   ```
3. In these files, a test is to be implemented if its body contains `TODO()`. Leave all other tests untouched.

### 2. Understand the context

Before implementing a test file:

1. Read the **production class** under test completely, plus the signatures of its collaborators (constructor parameters) and the domain classes used in its methods.
2. Look at **2–3 existing, implemented tests** in the same module and follow their conventions over the defaults below: how mocks are declared, existing test fixtures/builders/object mothers, which Kotest matchers are used, coroutine test setup.
3. Check the dependencies in the build file (`build.gradle.kts` / `pom.xml`): MockK, Kotest assertions, `kotlinx-coroutines-test`. Only use what is available. Do not add dependencies without asking the user.

### 3. Set up the test class

Default (if the repo has no other convention):

- Mocks and the system under test as **fields of the test class**:
  ```kotlin
  private val customerRepository = mockk<CustomerRepository>()
  private val sut = OrderService(customerRepository)
  ```
- JUnit 5 creates a new test class instance per test method, so fields are fresh for every test and no `clearMocks` is needed. If the class uses `@TestInstance(Lifecycle.PER_CLASS)`, reset the mocks in `@BeforeEach` instead.
- Use **strict mocks** (no `relaxed = true`). Stub only what the test needs. Use `relaxUnitFun = true` or `just Runs` only for `Unit` functions whose call is irrelevant to the test.
- Integration tests: use **real instances** of the classes that work together and mock only the system boundaries (repositories, HTTP clients, message producers, clocks). If a Spring context is required, follow the existing integration tests in the repo.

### 4. Implement each test

Keep the existing structure: the three comments stay, the code goes underneath, `TODO()` is removed.

**`// arrange`**
- Create only the test data relevant to this test. Use named arguments.
- Reuse existing fixtures/builders. If the same test data is needed in two or more tests and no fixture exists, add a small `private fun` at the end of the test class (e.g. `private fun anOrder(status: OrderStatus = OrderStatus.CREATED) = ...`).
- Stub collaborators: `every { ... } returns ...`, `answers { firstArg() }`, `throws ...`; for `suspend` functions `coEvery`.

**`// act`**
- Exactly **one** call of the method under test. Store the result in `val result` if there is one.
- For exception tests the call goes inside `shouldThrow`:
  ```kotlin
  // act
  val exception = shouldThrow<CustomerNotFoundException> { sut.createOrder(1L, items) }
  ```

**`// assert`**
- Assert exactly the behavior the test name describes. No unrelated extra assertions.
- Kotest matchers: `shouldBe`, `shouldNotBe`, `shouldBeNull`, `shouldNotBeNull`, `shouldContainExactly`, `shouldHaveSize`, `shouldBeEmpty`, `shouldContain`. Exceptions: `shouldThrow<T> { }` / `shouldThrowExactly<T> { }` from `io.kotest.assertions.throwables`.
- For exception tests, assert the message or relevant properties of the exception under `// assert` when the production code defines them.
- Interactions: `verify(exactly = 1) { ... }`, for "not called" `verify(exactly = 0) { ... }`, for `suspend` functions `coVerify`. Use `slot<T>()` with `capture(...)` to assert on arguments passed to a collaborator.
- Do not verify stubbed calls whose results are already asserted. Verify only when the interaction itself is the behavior under test (side effects).

**Coroutines:** if the method under test is `suspend`, wrap the test body in `runTest { }` (`kotlinx.coroutines.test.runTest`), keeping the three comments inside.

**Imports:** add the required imports explicitly (no wildcard imports, unless the repo uses them).

### 5. When a test does not fit the code

- **The name describes behavior the code does not have** (wrong definition): do not rename the test and do not invent a workaround. Leave `TODO()` in that test and list it in the report.
- **The test fails because the production code seems wrong** (possible bug): do not change production code and do not weaken the assertion to match the current behavior. List it in the report and ask the user how to proceed.
- **Missing test cases** you notice: do not add them; suggest them in the report.

### 6. Verify

1. Detect the build tool (`gradlew` or `mvnw`).
2. Run **only the affected test classes**, e.g.
   ```bash
   ./gradlew :<module>:test --tests "com.example.order.OrderServiceTest"
   ./mvnw -pl <module> test -Dtest=OrderServiceTest
   ```
3. Every implemented test must be green. Fix failures in the **test code**; see step 5 for failures caused by production code.
4. If the project has ktlint or detekt configured, run it for the affected modules and fix findings in the test code.
5. Make sure no `TODO()` remains in the implemented tests (except those listed in the report per step 5).

### 7. Final report

At the end, briefly output:
- per test file: number of implemented tests and test run result
- tests left with `TODO()` and why
- failing tests caused by production code, with a description of the suspected bug
- suggested additional test cases

## Example

Production code (from the define-tests example):

```kotlin
class CustomerNotFoundException(customerId: Long) : RuntimeException("Customer $customerId not found")

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
}
```

Implemented test file:

```kotlin
package com.example.order

import io.kotest.assertions.throwables.shouldThrow
import io.kotest.matchers.shouldBe
import io.mockk.every
import io.mockk.mockk
import io.mockk.slot
import io.mockk.verify
import org.junit.jupiter.api.Test

class OrderServiceTest {

    private val customerRepository = mockk<CustomerRepository>()
    private val orderRepository = mockk<OrderRepository>()
    private val sut = OrderService(customerRepository, orderRepository)

    // --- createOrder ---

    @Test
    fun `createOrder should save order with status CREATED when customer exists`() {
        // arrange
        val customer = aCustomer()
        val items = listOf(anItem())
        every { customerRepository.findById(1L) } returns customer
        every { orderRepository.save(any()) } answers { firstArg() }

        // act
        val result = sut.createOrder(1L, items)

        // assert
        result.status shouldBe OrderStatus.CREATED
        result.customer shouldBe customer
        result.items shouldBe items
    }

    @Test
    fun `createOrder should throw CustomerNotFoundException when customer does not exist`() {
        // arrange
        every { customerRepository.findById(1L) } returns null

        // act
        val exception = shouldThrow<CustomerNotFoundException> {
            sut.createOrder(1L, listOf(anItem()))
        }

        // assert
        exception.message shouldBe "Customer 1 not found"
    }

    @Test
    fun `createOrder should throw IllegalArgumentException when items are empty`() {
        // arrange
        val items = emptyList<Item>()

        // act
        val exception = shouldThrow<IllegalArgumentException> {
            sut.createOrder(1L, items)
        }

        // assert
        exception.message shouldBe "items must not be empty"
    }

    @Test
    fun `createOrder should not save order when customer does not exist`() {
        // arrange
        every { customerRepository.findById(1L) } returns null

        // act
        shouldThrow<CustomerNotFoundException> { sut.createOrder(1L, listOf(anItem())) }

        // assert
        verify(exactly = 0) { orderRepository.save(any()) }
    }

    // --- cancelOrder ---

    @Test
    fun `cancelOrder should save order with status CANCELLED when order is not shipped`() {
        // arrange
        val saved = slot<Order>()
        every { orderRepository.getById(42L) } returns anOrder(status = OrderStatus.CREATED)
        every { orderRepository.save(capture(saved)) } answers { firstArg() }

        // act
        sut.cancelOrder(42L)

        // assert
        saved.captured.status shouldBe OrderStatus.CANCELLED
    }

    @Test
    fun `cancelOrder should throw IllegalStateException when order is already shipped`() {
        // arrange
        every { orderRepository.getById(42L) } returns anOrder(status = OrderStatus.SHIPPED)

        // act
        val exception = shouldThrow<IllegalStateException> { sut.cancelOrder(42L) }

        // assert
        exception.message shouldBe "shipped orders cannot be cancelled"
    }

    private fun aCustomer() = Customer(id = 1L, name = "Jane Doe")

    private fun anItem() = Item(sku = "A-1", quantity = 1)

    private fun anOrder(status: OrderStatus) =
        Order(customer = aCustomer(), items = listOf(anItem()), status = status)
}
```

Notes on the example:
- The empty-items test stubs nothing: `require` fails before any collaborator is called, and the strict mocks would fail the test if one were.
- The happy-path test does not `verify` the `save` call, because its result is already asserted. The "should not save" test verifies it, because the missing interaction is the behavior under test.