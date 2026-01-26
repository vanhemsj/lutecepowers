# Phase 14: Test Migration (JUnit 4 → JUnit 5) & Final Review

## Test dependency — MANDATORY

Tests extending `LuteceTestCase` require `library-lutece-unit-testing`. The global-pom `8.0.0-SNAPSHOT` manages the version — just declare it:

```xml
<dependency>
    <groupId>fr.paris.lutece.plugins</groupId>
    <artifactId>library-lutece-unit-testing</artifactId>
    <type>jar</type>
    <scope>test</scope>
</dependency>
```

If this dependency is missing, tests will fail with `cannot find symbol: class LuteceTestCase`.

## Annotation changes

| JUnit 4 | JUnit 5 |
|---------|---------|
| `import org.junit.Test` | `import org.junit.jupiter.api.Test` |
| `import org.junit.Before` | `import org.junit.jupiter.api.BeforeEach` |
| `import org.junit.After` | `import org.junit.jupiter.api.AfterEach` |
| `@Before` | `@BeforeEach` |
| `@After` | `@AfterEach` |
| `@BeforeClass` | `@BeforeAll` |
| `@AfterClass` | `@AfterAll` |

## Assertion parameter order change

```java
// JUnit 4
assertEquals("Message", expected, actual);
// JUnit 5
assertEquals(expected, actual, "Message");
```

## Mock class renames

| v7 | v8 |
|----|-----|
| `MokeHttpServletRequest` | `MockHttpServletRequest` |
| `request.addMokeHeader(name, value)` | `request.addHeader(name, value)` |

## Assertion style (JUnit 5)

```java
// v7: static import
import static org.junit.Assert.*;
assertTrue(condition);

// v8: use Assertions class
import org.junit.jupiter.api.Assertions;
Assertions.assertTrue(condition);
```

## CDI Test Extension (for dynamic bean registration in tests)

Create `src/test/resources/META-INF/services/jakarta.enterprise.inject.spi.Extension` listing your test extension class. The extension can dynamically register mock beans:

```java
public class MyTestExtension implements Extension {
    protected void addBeans(@Observes AfterBeanDiscovery abd, BeanManager bm) {
        abd.addBean()
            .beanClass(MockService.class)
            .name("mockBeanName")
            .addTypes(MockService.class, IService.class)
            .addQualifier(NamedLiteral.of("mockBeanName"))
            .scope(ApplicationScoped.class)
            .produceWith(obj -> new MockService());
    }
}
```

## Verification (MANDATORY — final phase)

1. Run: `mvn clean install` (with tests — this is the final build)
2. If BUILD FAILURE or TEST FAILURE → fix test errors, re-run until SUCCESS
3. Verify no JUnit 4 imports remain: `grep -r "org.junit.Test\b" src/test/` → must return nothing (only `org.junit.jupiter`)
4. Mark task as completed ONLY when all tests pass and `mvn clean install` succeeds

## V8 Compliance Review — MANDATORY

After the final build succeeds, launch the **lutece-v8-reviewer** agent to verify the migration is complete:

> Delegate to the `lutece-v8-reviewer` agent to review the project for v8 compliance.

### Handling reviewer results

1. **FAIL items** → You MUST fix every FAIL item. These are migration errors that will cause runtime problems.
2. **WARN items** → You MUST attempt to fix every WARN item. Most warnings indicate incomplete migration patterns that should be corrected. Only skip a WARN fix if it is technically impossible or would break existing functionality — in that case, document why in the phase report.
3. After fixing FAIL and WARN items, re-run `mvn clean install` to ensure the build still passes.
4. If new issues appear after fixes, re-launch the reviewer agent and repeat until clean.
5. **Do NOT mark this phase as completed until all FAIL items are resolved and all fixable WARN items are addressed.**
