# Phase 13: First Build Checkpoint, SQL Migration & Full Grep Verification

Run these checks on the entire project:

- [ ] `grep -r "org.springframework" src/` → **must return nothing**
- [ ] `grep -r "javax.servlet" src/` → **must return nothing**
- [ ] `grep -r "javax.validation" src/` → **must return nothing**
- [ ] `grep -r "javax.annotation.PostConstruct\|javax.annotation.PreDestroy" src/` → **must return nothing**
- [ ] `grep -r "javax.inject" src/` → **must return nothing**
- [ ] `grep -r "javax.ws.rs" src/` → **must return nothing**
- [ ] `grep -r "SpringContextService" src/` → **must return nothing**
- [ ] `grep -r "net.sf.ehcache" src/` → **must return nothing**
- [ ] `grep -r "_context.xml" webapp/` → **no context XML files remain**
- [ ] `beans.xml` exists at correct location
- [ ] All DAO classes have `@ApplicationScoped`
- [ ] All Service classes have `@ApplicationScoped`
- [ ] All `getInstance()` methods use `CDI.current().select()` if kept
- [ ] No `final` keyword on any DAO, Service, or Home class (CDI cannot proxy final classes, and v8 convention removes `final` from these layers)
- [ ] Project install: `mvn clean install`

## SQL Migration

### Liquibase headers for ALL SQL scripts
**Every** SQL file in the project must start with Liquibase headers — this includes:
- `create_db_*.sql` (table creation)
- `init_core_*.sql` (core data initialization)
- `update_db_*.sql` (upgrade scripts)

```sql
-- liquibase formatted sql
-- changeset pluginName:script_name.sql
-- preconditions onFail:MARK_RAN onError:WARN
```

Add these headers to **all existing SQL files**, not just new ones.

### Upgrade scripts
Create upgrade SQL scripts: `update_db_pluginName-oldVersion-newVersion.sql`

## Verification (MANDATORY before next phase)

1. Run ALL grep checks listed above — every single one must return nothing
2. Run: `mvn clean install -Dmaven.test.skip=true`
3. If BUILD FAILURE → fix all remaining errors, re-run until SUCCESS
4. Mark task as completed ONLY when ALL grep checks pass AND build succeeds
