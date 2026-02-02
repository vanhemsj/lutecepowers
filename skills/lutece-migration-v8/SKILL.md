---
name: lutece-migration-v8
description: "Migration guide v7 → v8 (Spring → CDI/Jakarta). Task-based execution with blockedBy dependencies and mandatory verification per phase."
---

# Lutece Migration v7 to v8 - Orchestrator

## Purpose

This skill migrates any Lutece plugin/module/library from v7 to v8. It uses **task-based phase execution** with `blockedBy` dependencies to enforce order and mandatory verification between phases.

**IMPORTANT:** Each phase's detailed instructions are in a separate file. You MUST read the phase file before starting each phase.

**MIGRATION SAMPLES:** Real migration examples (v7 → v8 diffs with analysis) are available at `${CLAUDE_PLUGIN_ROOT}/migrations-samples/`. Consult them when needed to clarify how a specific migration pattern was applied in practice.

---

## EXECUTION STRATEGY (MANDATORY)

### Task-Based Phase Execution

The AI MUST create one task per phase using `TaskCreate` before starting any work. Each task has explicit `blockedBy` dependencies. The AI MUST NOT start a phase until all blocking phases are completed.

### Step 1 — Create all 15 tasks

```
TaskCreate({ subject: "Phase 0: Analysis", description: "Read phases/phase-0-analysis.md then execute ALL steps. Output: migration impact report.", activeForm: "Analyzing project..." })

TaskCreate({ subject: "Phase 1: POM migration", description: "Read phases/phase-1-pom.md then execute ALL steps. Verify: POM content checks (no build).", activeForm: "Migrating POM..." })

TaskCreate({ subject: "Phase 2: javax→jakarta", description: "Read phases/phase-2-javax-jakarta.md then execute ALL steps. Verify: grep checks (no build).", activeForm: "Migrating javax→jakarta..." })

TaskCreate({ subject: "Phase 3: Spring→CDI", description: "Read phases/phase-3-spring-cdi.md then execute ALL steps. Verify: grep checks (no build).", activeForm: "Migrating Spring→CDI..." })

TaskCreate({ subject: "Phase 4: Events", description: "Read phases/phase-4-events.md then execute ALL steps. Verify: grep checks (no build).", activeForm: "Migrating events..." })

TaskCreate({ subject: "Phase 5: Cache", description: "Read phases/phase-5-cache.md then execute ALL steps. Verify: grep checks (no build).", activeForm: "Migrating cache..." })

TaskCreate({ subject: "Phase 6: Config", description: "Read phases/phase-6-config.md then execute ALL steps. Verify: visual check (no build).", activeForm: "Migrating config..." })

TaskCreate({ subject: "Phase 7: REST", description: "Read phases/phase-7-rest.md then execute ALL steps. Verify: grep checks (no build).", activeForm: "Migrating REST..." })

TaskCreate({ subject: "Phase 8: web.xml & plugin descriptor", description: "Read phases/phase-8-webxml.md then execute ALL steps. Update web.xml namespace + plugin.xml (version, min-core-version, remove application-class). Verify: grep checks (no build).", activeForm: "Migrating web.xml & plugin descriptor..." })

TaskCreate({ subject: "Phase 9: JspBean migration", description: "Read phases/phase-9-jspbean.md then execute ALL steps. Verify: grep checks (no build).", activeForm: "Migrating JspBeans..." })

TaskCreate({ subject: "Phase 10: JSP migration", description: "Read phases/phase-10-jsp.md then execute ALL steps. Verify: grep checks (no build).", activeForm: "Migrating JSPs..." })

TaskCreate({ subject: "Phase 11: Templates", description: "Read phases/phase-11-templates.md then execute ALL steps. Rewrite admin templates to v8 macros, migrate skin templates, remove jQuery. Verify: grep checks + visual check (no build).", activeForm: "Migrating templates..." })

TaskCreate({ subject: "Phase 12: Logging", description: "Read phases/phase-12-logging.md then execute ALL steps. Verify: grep checks (no build).", activeForm: "Migrating logging..." })

TaskCreate({ subject: "Phase 13: First build checkpoint & SQL", description: "Read phases/phase-13-build-checkpoint.md then execute ALL steps. FIRST BUILD: mvn clean install -Dmaven.test.skip=true + SQL migration.", activeForm: "Running first build checkpoint..." })

TaskCreate({ subject: "Phase 14: Tests & final review", description: "Read phases/phase-14-tests.md then execute ALL steps. Final build: mvn clean install (WITH tests) + launch lutece-v8-reviewer agent.", activeForm: "Migrating tests & final review..." })
```

### Step 2 — Wire dependencies (use actual task IDs returned by TaskCreate)

```
TaskUpdate({ taskId: "<phase1_id>",  addBlockedBy: ["<phase0_id>"] })
TaskUpdate({ taskId: "<phase2_id>",  addBlockedBy: ["<phase1_id>"] })
TaskUpdate({ taskId: "<phase3_id>",  addBlockedBy: ["<phase2_id>"] })
TaskUpdate({ taskId: "<phase4_id>",  addBlockedBy: ["<phase3_id>"] })
TaskUpdate({ taskId: "<phase5_id>",  addBlockedBy: ["<phase3_id>"] })
TaskUpdate({ taskId: "<phase6_id>",  addBlockedBy: ["<phase3_id>"] })
TaskUpdate({ taskId: "<phase7_id>",  addBlockedBy: ["<phase3_id>"] })
TaskUpdate({ taskId: "<phase8_id>",  addBlockedBy: ["<phase3_id>"] })
TaskUpdate({ taskId: "<phase9_id>",  addBlockedBy: ["<phase4_id>", "<phase5_id>", "<phase6_id>", "<phase7_id>", "<phase8_id>"] })
TaskUpdate({ taskId: "<phase10_id>", addBlockedBy: ["<phase9_id>"] })
TaskUpdate({ taskId: "<phase11_id>", addBlockedBy: ["<phase10_id>"] })
TaskUpdate({ taskId: "<phase12_id>", addBlockedBy: ["<phase11_id>"] })
TaskUpdate({ taskId: "<phase13_id>", addBlockedBy: ["<phase12_id>"] })
TaskUpdate({ taskId: "<phase14_id>", addBlockedBy: ["<phase13_id>"] })
```

Phases 4-8 share the same blockedBy (Phase 3) and unblock independently. Phases 9-12 (JspBean → JSP → Templates → Logging) run sequentially after.

---

## Phase Execution Protocol

For each phase:
1. **Read the phase file**: `Read` the file `phases/phase-N-name.md` from this skill's directory
2. `TaskUpdate` → set status to `in_progress`
3. Execute ALL steps described in the phase file
4. Run the **mandatory verification** described at the end of the phase file
5. If verification fails → fix errors, re-run verification until it passes
6. `TaskUpdate` → set status to `completed` ONLY when verification passes
7. Output the phase report (see format below)
8. Check `TaskList` → pick next unblocked task → go to step 1

---

## Build Verification Strategy

The project **will NOT compile** between Phase 1 and Phase 13. This is expected — each phase fixes one aspect while others remain broken.

| Phases | Verification type |
|--------|------------------|
| 0 | Impact report completeness |
| 1-12 | **Grep checks only** (no build) |
| 13 | **First build checkpoint**: `mvn clean install -Dmaven.test.skip=true` — must pass |
| 14 | **Full build with tests**: `mvn clean install` + **lutece-v8-reviewer** agent |

---

## Strict Rules

- **Phases 1-12: verify with grep checks only** — do NOT run `mvn install` (it will fail)
- **Phase 13+: NEVER mark a task completed without BUILD SUCCESS**
- **NEVER start a phase while a blocking phase is still `in_progress` or `pending`**
- **If verification fails, you MUST fix all errors before re-running verification**
- **After each phase, report: files modified, grep check results, build status (if applicable)**
- **No commits between phases** — commit only at the end when everything is green
- **ALWAYS re-read the phase file** before starting a phase — do not rely on context memory
- **Source lookup: ALWAYS use `~/.lutece-references/`** to read Lutece dependency sources (Read/Grep/Glob). NEVER decompile jars from `.m2/repository/`. The reference repos are cloned at session start for exactly this purpose

---

## Phase Report Format

After each phase, output:
```
## Phase N Complete
- Files modified: [list]
- Grep checks: [PASS/FAIL with details]
- Build: [SUCCESS/FAILURE or N/A for phases 1-12]
- Task status: completed
```

---

## Key Imports Reference

```java
// CDI Core
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.context.RequestScoped;
import jakarta.enterprise.inject.spi.CDI;
import jakarta.enterprise.inject.Instance;
import jakarta.enterprise.inject.Produces;
import jakarta.enterprise.inject.Alternative;
import jakarta.enterprise.inject.literal.NamedLiteral;
import jakarta.inject.Inject;
import jakarta.inject.Named;
import jakarta.inject.Singleton;

// CDI Events
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.context.Initialized;
import jakarta.annotation.Priority;

// Lifecycle
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

// Servlet
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;

// REST
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;

// Validation
import jakarta.validation.ConstraintViolation;

// Config
import org.eclipse.microprofile.config.inject.ConfigProperty;

// Cache
import javax.cache.Cache; // NOTE: javax, NOT jakarta
```

---

## Common Migration Patterns

### Pattern: Static DAO in Home class
```java
// v7
private static IMyDAO _dao = SpringContextService.getBean("myDAO");
// v8
private static IMyDAO _dao = CDI.current().select(IMyDAO.class).get();
```

### Pattern: Named bean lookup
```java
// v7
IProvider p = SpringContextService.getBean("myNamedProvider");
// v8
IProvider p = CDI.current().select(IProvider.class, NamedLiteral.of("myNamedProvider")).get();
```

### Pattern: Iterate beans of type
```java
// v7
List<IProvider> list = SpringContextService.getBeansOfType(IProvider.class);
// v8
Instance<IProvider> list = CDI.current().select(IProvider.class);
```

### Pattern: Singleton service
```java
// v7
public final class MyService {
    private static MyService _instance;
    public static synchronized MyService getInstance() { ... }
}
// v8
@ApplicationScoped
public class MyService {
    public static MyService getInstance() {
        return CDI.current().select(MyService.class).get();
    }
}
```

### Pattern: Spring XML complex bean → CDI Producer
```java
// v7 XML: <bean id="x" class="Y"><property name="p" value="v"/></bean>
// v8:
@ApplicationScoped
public class YProducer {
    @Produces @Named("x") @ApplicationScoped
    public Y produce() {
        Y y = new Y();
        y.setP("v");
        return y;
    }
}
```
