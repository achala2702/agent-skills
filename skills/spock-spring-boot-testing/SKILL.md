---
name: spock-spring-boot-testing
description: Use this skill whenever writing, reviewing, or refactoring unit or integration tests for a Spring Boot application using the Spock Framework (Groovy Specification classes) combined with spring-boot-starter-test. Trigger for requests like "write a Spock test for X", "convert this JUnit test to Spock", "add a Spock spec for this controller/service/repository", "set up Testcontainers with Spock", "why is my Spock test failing / mocks not injecting", or any mention of *.groovy Specification classes, @SpringBean, given/when/then/where blocks, or Spock+JaCoCo/CI configuration. Encodes current (2024-2026) industry-aligned conventions so tests are written consistently without re-deriving best practices each time.
---

# Spock + Spring Boot Testing

Reference guide for writing idiomatic Spock (Groovy) tests in Spring Boot projects, covering test architecture, mocking strategy, data-driven testing, Testcontainers, and build/CI configuration.

## When to use this skill

Use this any time the task is to:
- Write a new `*Spec` class (unit, slice, or full integration test) for a Spring Boot component
- Convert an existing JUnit/Mockito test to Spock
- Review a Spock spec for adherence to conventions (fixture lifecycle, mocking choice, `where:` table structure)
- Diagnose common Spock+Spring failures (null autowired beans, context caching blowup, container start-order errors, Surefire not picking up specs)
- Set up or fix build/CI configuration for Spock (Maven GMavenPlus/Surefire, Gradle Groovy plugin, JaCoCo)

## Version baseline

- **Spock 2.4-groovy-4.0** (GA 2025-12-11) on **Spring Boot 3.x**. Runs on the JUnit 5 Platform — no Sputnik runner.
- If the project pins an older Spock/Groovy combo, match examples below to that version but flag anything that depends on 2.x-only behavior (default-unrolling, JUnit Platform).
- `@Unroll` is **default-on** for parameterized features in Spock 2.x — do not add bare `@Unroll` to new specs; only use `@Unroll("custom pattern")` for a custom name, or `@Rollup` to disable unrolling for one feature/spec.

## Decision guide: which kind of spec to write

Ask "what's the narrowest slice that exercises the behavior under test?" in this order:

1. **Pure unit spec** — plain `Specification`, no Spring annotations, constructor-injected collaborators as `Mock()`/`Stub()`. Use for service/domain logic with no framework wiring needed.
2. **Test slice spec** — `@WebMvcTest`, `@DataJpaTest`, `@JsonTest`, `@JdbcTest`, etc. on a `Specification`. Use for controller-layer or persistence-layer behavior that needs partial Spring context.
3. **Full `@SpringBootTest` integration spec** — only when you need the whole wired application (multiple layers together, real transaction boundaries, Testcontainers-backed DB). Keep these the minority of the suite — they're the slowest and most context-cache-sensitive.

If in doubt, prefer the narrower slice. A test suite dominated by `@SpringBootTest` specs is a smell (slow, few isolated failures, context-cache churn).

## Dependency & build setup

### Gradle (Groovy DSL)

```groovy
plugins {
    id 'java'
    id 'groovy'
    id 'org.springframework.boot' version '3.4.0'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'jacoco'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.spockframework:spock-core:2.4-groovy-4.0'
    testImplementation 'org.spockframework:spock-spring:2.4-groovy-4.0'
    testImplementation 'org.testcontainers:spock'
}

test { useJUnitPlatform() }

jacoco { toolVersion = "0.8.12" }
jacocoTestReport {
    reports { xml.required = true; html.required = true }
    afterEvaluate {
        classDirectories.setFrom(classDirectories.files.collect {
            fileTree(dir: it, exclude: ['**/*Spec.class', '**/dto/**', '**/*_closure*'])
        })
    }
}
```

Specs live under `src/test/groovy`. No GMavenPlus equivalent needed — the `groovy` plugin handles compilation.

### Maven

Requires **GMavenPlus** (Groovy compilation) and a **Surefire include** for `*Spec` (Surefire's defaults are `**/Test*`, `**/*Test`, `**/*Tests`, `**/*TestCase` — `*Spec` is silently skipped otherwise).

```xml
<dependencies>
  <dependency>
    <groupId>org.spockframework</groupId>
    <artifactId>spock-core</artifactId>
    <version>2.4-groovy-4.0</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.spockframework</groupId>
    <artifactId>spock-spring</artifactId>
    <version>2.4-groovy-4.0</version>
    <scope>test</scope>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <groupId>org.codehaus.gmavenplus</groupId>
      <artifactId>gmavenplus-plugin</artifactId>
      <version>4.2.1</version>
      <executions>
        <execution>
          <goals><goal>compile</goal><goal>compileTests</goal></goals>
        </execution>
      </executions>
    </plugin>
    <plugin>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.5.4</version>
      <configuration>
        <useFile>false</useFile>
        <includes>
          <include>**/*Test</include>
          <include>**/*Spec</include>
        </includes>
      </configuration>
    </plugin>
  </plugins>
</build>
```

Add `jacoco-maven-plugin` with `<exclude>` patterns for `*Spec`, DTOs, and synthetic/closure classes — JaCoCo misreports Groovy closures as uncovered methods and adds a phantom "missed" line at the end of value-returning methods; this is known tool noise, not a real coverage gap.

**Symptom → cause table:**
| Symptom | Cause |
|---|---|
| Specs never run in Maven | Missing `**/*Spec` in Surefire includes |
| Gradle reports "0 tests" | Missing `useJUnitPlatform()` |
| `SpockTransform` instantiation error | Groovy/Gradle Groovy-plugin version mismatch |
| Coverage looks artificially low | Groovy synthetic/closure methods + end-of-method phantom line in JaCoCo — filter, don't chase to 100% |

## Naming conventions

- Specification classes end in **`*Spec`** (e.g. `AccountServiceSpec`); full integration specs often `*SpecIT` or `*IT`.
- Feature methods are **String literals read as sentences**: `def "returns 404 when the account does not exist"()`.
- Data-driven method names embed `#variable` placeholders so each unrolled iteration reports readably: `def "adding #a and #b yields #expected"()`.

## Block structure (given/when/then/expect/where)

- `given:` (alias `setup:`) — preconditions. Prefer `given:` over `setup:` for readability.
- `when:` / `then:` — always paired; `when:` may contain arbitrary code, `then:` is restricted to conditions, exception conditions, interactions, and variable definitions. Multiple when/then pairs are fine in one feature.
- `expect:` — combined stimulus+assertion; ideal for pure functions.
- `and:` — subdivides any block, purely for readability, no semantics.
- `cleanup:` — always runs, like a `finally`.
- `where:` — supplies parameterized data (see Data-driven section).

Conditions are plain boolean expressions — no assertion API needed; Spock's power-assert renders every sub-expression on failure. To assert outside then/expect blocks, use Groovy's `assert` keyword. Use `with(obj) { ... }` or `verifyAll { ... }` to group related assertions on one object.

```groovy
class CalculatorSpec extends Specification {

    def "adding #a and #b yields #expected"() {
        given: "a calculator"
        def calc = new Calculator()

        when: "the numbers are added"
        def result = calc.add(a, b)

        then: "the sum is correct"
        result == expected

        where:
        a  | b || expected
        1  | 2 || 3
        -5 | 2 || -3
    }
}
```

## Mocking strategy

**Default to Spock's native `Mock()` / `Stub()` / `Spy()`** — don't reach for Mockito inside a Spock spec unless the team standard requires it.

- `Mock()` — verifies interactions AND can stub return values. Use when the collaboration itself matters.
- `Stub()` — only provides canned responses, cannot be verified with `1 *`. Use it to *communicate* "this collaborator's calls don't matter, only its outputs do."
- `Spy()` — wraps a real object, delegating unstubbed calls. Use sparingly; if you need a spy often, reconsider the design under test.

### Interaction syntax

```groovy
1 * subscriber.receive("hello")        // exactly once, exact argument
(1..3) * subscriber.receive(_)         // 1 to 3 times, any single argument
0 * subscriber.receive(_)              // never called
1 * service.add(3, 5)                  // exact arguments, exact call count
```

Stub return values with `>>`:

```groovy
service.call(_) >> "fixed value"                          // constant
service.call(_) >>> ["first", "second", "third"]          // sequence across calls (no behavior/exceptions allowed here)
service.call(_) >> { args -> args[0].size() > 3 ? "ok" : "fail" }   // computed from arguments
service.call(_) >> { throw new RuntimeException("boom") }  // throw
service.call(_) >> 'first' >> { generatorMethod() } >> 'third' >> { throw new EmptyStackException() } // chained mixed behavior
```

Declare stubbing near mock creation or in `given:`; declare verification interactions in `then:` (they take precedence over prior stubbing there).

### In a Spring context: `@SpringBean` / `@SpringSpy`, not `@MockBean`

```groovy
@WebMvcTest(GreetingController)
class GreetingControllerSpec extends Specification {

    @Autowired MockMvc mvc

    @SpringBean
    HelloWorldService helloWorldService = Stub()

    def "GET / returns the greeting"() {
        given:
        helloWorldService.helloMessage >> "hello world"

        expect:
        mvc.perform(get("/"))
           .andExpect(status().isOk())
           .andExpect(content().string("hello world"))
    }
}
```

Rules:
- The field must be **strongly typed** (never `def`/`Object`) and assigned its `Mock()`/`Stub()`/`Spy()` directly in the initializer.
- `@SpringSpy` wraps an existing bean instead (no initializer).
- `@StubBeans(Foo, Bar)` registers plain stubs for dependencies you only need to satisfy, not interact with.
- **Never put a mock/stub in a `static` or `@Shared` field** — mocks are bound to a single spec instance.
- `@SpringBean` mutates the `ApplicationContext`, creating a spec-unique context that Spring's test context caching cannot reuse elsewhere — minimize the number of distinct `@SpringBean`/profile/property combinations across the suite to avoid slow builds.

**When Mockito is still acceptable:** existing team standard already uses `@MockitoBean`/`@MockitoSpyBean` (the current replacements for the now-deprecated `@MockBean`/`@SpyBean`, deprecated since Spring Boot 3.4.0), or a third-party integration expects Mockito mock objects specifically. Mixing Mockito into a Spock spec forfeits Spock's interaction DSL (`1 *`, `>>`), so treat it as an exception, not the default.

## Data-driven testing (`where:` tables)

- Iterations unroll automatically in Spock 2.x — no `@Unroll` needed unless customizing the name pattern.
- Separate inputs from expected outputs with `||` (visual only).
- Tables need ≥2 columns; use a `_` filler column if you only have one real input.
- Use derived variables (`c = a + b`) or data pipes (`a << [1, 2, 3]`) to reduce duplication; map deconstruction is supported: `where: [a, b, c] << [[a: 1, b: 3, c: 5], ...]`.

**Do:**
```groovy
where:
role      | permissions       || canDelete
"admin"   | ["delete"]        || true
"viewer"  | []                || false
```

**Avoid:**
- Wide tables (6+ columns) that scroll off-screen — split into multiple features or use a domain object built via a helper method instead of columns-of-primitives.
- Tables mixing unrelated concerns just to save a feature method — one behavior per feature.
- A single-column table without a `_` filler (compile error) or one so large the intent is unreadable at a glance — extract a builder or use `where:` with a `<<` pipe from a named list instead.

## Test slices inside Groovy specs

All `@BootstrapWith`-based slice annotations (`@WebMvcTest`, `@DataJpaTest`, `@JsonTest`, `@JdbcTest`, `@DataMongoTest`, …) work directly on `Specification` classes via `spock-spring`.

```groovy
@DataJpaTest
class JokeRepositorySpec extends Specification {

    @Autowired JokeRepository jokeRepository
    @Autowired TestEntityManager entityManager

    def "finds joke by category"() {
        given:
        entityManager.persist(new Joke(category: "pun", text: "why did the chicken..."))

        expect:
        jokeRepository.findByCategory("pun").size() == 1
    }
}
```

Caveat: forgetting the `spock-spring` dependency is the #1 cause of `null` `@Autowired` fields — the context silently never loads.

## Full integration test

```groovy
@SpringBootTest
class MovieServiceIntegrationSpec extends Specification {

    @Autowired MovieRepository movieRepository
    @Autowired MovieService movieService

    def "saved movie can be found in the database"() {
        given:
        def movie = new Movie(title: "Wrath of Khan")

        when:
        movieService.save(movie)

        then:
        movieRepository.findByTitle("Wrath of Khan")
    }
}
```

## Testcontainers integration

Add `org.testcontainers:spock`. For a Spring-wired test, declare the container as a **`@Shared static`** field and start it manually inside a static `@DynamicPropertySource` method (Spring requires that method to be static, and the container must already be running when it's read):

```groovy
@Testcontainers
@SpringBootTest
class SampleIntegrationSpec extends Specification {

    @Shared
    static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:6")

    @DynamicPropertySource
    static void mongoProps(DynamicPropertyRegistry registry) {
        mongoDBContainer.start()
        registry.add("spring.data.mongodb.uri", () -> mongoDBContainer.replicaSetUrl)
    }
}
```

- Prefer Spring Boot 3.1+ **`@ServiceConnection`** on a container bean when available — it removes most `@DynamicPropertySource` boilerplate.
- For a **singleton container shared across multiple spec classes**, start it in a static initializer on an abstract base spec and do *not* combine that with `@Testcontainers`/`@Container` lifecycle annotations — those stop containers at the end of each test class, which will kill your "shared" container prematurely.

## Fixture lifecycle & isolation pitfalls

- `setup()` / `cleanup()` run before/after **every** feature method. `setupSpec()` / `cleanupSpec()` run **once per class**.
- Prefer normal instance fields for per-feature state — they're reset automatically between features (Spock creates a fresh spec instance per feature).
- `@Shared` fields persist across all features in a spec — use only for genuinely expensive, intentionally-shared objects (a `DataSource`, a started container). Never store per-test mutable state or **mocks** in `@Shared`/`static` fields — mocks are bound to one spec instance and sharing them causes broken or misleading verification.
- A cached Spring context carries over mutable bean state and DB rows between specs that share configuration. Prefer `@Transactional` rollback (default in `@DataJpaTest`) over `@DirtiesContext`, which evicts the cache and slows the whole suite — use it only as a last resort.
- Groovy's dynamic typing means a typo or an unexpectedly "truthy" return value can silently pass a `then:`/`expect:` condition — type collaborator fields explicitly (required anyway for `@SpringBean`) and use `with()`/`verifyAll()` for structured assertions instead of loose chained conditions.

## Coverage expectations

Aim for **~80% line/branch coverage** on the code under test, driven by scenario completeness rather than chasing the number directly. For each unit under test (method/endpoint/handler), plan feature methods across:

- **Main scenario** — the happy path / most common valid use case.
- **Other paths** — alternative valid inputs, boundary values, empty/null-but-valid collections, branches (`if`/`switch`/ternary) not hit by the main scenario. Prefer a `where:` table over separate features when the only thing changing is input/output data.
- **Exceptions/errors** — invalid input, failed preconditions, collaborator throws, mapped HTTP error responses. Assert via `thrown()`/`notThrown()` in `then:`, or the exact status/body for slice specs.

If a method has no alternative-path or error-path branches, don't invent scenarios to hit a percentage — note that coverage is expected to be near 100% for that unit and move on. Treat a coverage shortfall as a signal to check for an untested branch, not as license to pad the count with redundant happy-path variants.

### What to prioritize

When time/scope is limited, spend it here first — these carry the most risk if untested:

- **Business-critical paths** — code whose failure has direct business impact (e.g. payment processing, order validation, inventory reservation).
- **Complex algorithms** — non-trivial branching or calculation logic (e.g. pricing, discount/tax calculations, eligibility rules) where a subtle bug is easy to introduce and hard to spot by inspection.
- **Error handling** — exception paths, validation failures, and edge cases (nulls, empty collections, boundary values) that are easy to skip when only the happy path is exercised.
- **Integration points** — boundaries to external APIs, databases, message queues, or other services, where contract mismatches and failure modes (timeouts, 4xx/5xx, malformed responses) surface.

Simple getters/setters, DTOs/mappers with no logic, and framework boilerplate are low priority — don't spend feature methods on them just to move the coverage number.

## Checklist: writing a new Spock spec

1. **Pick the narrowest scope** — pure unit spec > test slice > full `@SpringBootTest` (see Decision guide).
2. **Name the class** `<Subject>Spec` and give each feature method a sentence-style String name.
3. **Structure blocks**: `given:` → `when:` → `then:`, or `expect:` for pure functions. One stimulus per when/then pair.
4. **Choose mocks**: Spock `Mock()`/`Stub()`/`Spy()` by default; inject into Spring context via `@SpringBean`/`@SpringSpy`/`@StubBeans`, not `@MockBean`.
5. **If parameterizing**, build a `where:` table — inputs `||` outputs, no `@Unroll` needed, keep columns few and focused.
6. **Cover scenario breadth**: main happy path, other valid/edge paths, and exception/error paths for the unit under test (see Coverage expectations) — don't stop at the happy path alone.
7. **If it needs infra** (DB, queue), reach for the matching test slice first; only use Testcontainers + `@SpringBootTest` if you truly need the wired app against a real backing service.
8. **Check fixture scope** — no mocks or mutable state in `@Shared`/static fields; use `setup()`/`cleanup()` for per-feature reset.
9. **Verify build wiring** — Maven: Surefire includes `**/*Spec`; Gradle: `useJUnitPlatform()` is set. Confirm the new spec actually runs, not just compiles.
10. **Sanity-check coverage output** if relevant — target ~80% and ignore JaCoCo's Groovy closure/end-of-method noise rather than writing tests to chase the number.
