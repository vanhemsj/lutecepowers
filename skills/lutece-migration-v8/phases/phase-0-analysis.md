# Phase 0: Analysis (Mandatory Before Any Code Change)

## Steps

1. Read the full project structure (`find . -type f -name "*.java" -o -name "*.xml" -o -name "*.html" -o -name "*.properties"`)
2. Identify the project type: **plugin**, **module**, or **library**
3. Read `pom.xml` completely
4. List all `*_context.xml` files
5. List all Java files with their current imports
6. List all template files (`.html`)
7. List all `.properties` files
8. Identify all Spring beans declared in context XML files
9. Identify all `SpringContextService.getBean()` calls
10. Identify all singleton patterns (`getInstance()`)
11. Identify all `javax.*` imports
12. Identify all cache usage (EhCache 2.x)
13. Identify all custom event/listener patterns
14. Identify all REST endpoints (JAX-RS)

## Dependency v8 Verification (BLOCKER)

For each **Lutece dependency** found in `pom.xml` (groupId `fr.paris.lutece.*`):

1. **Check `~/.lutece-references/`** — If the repo is already cloned, read its `pom.xml` to confirm `<parent><version>` is `8.0.0-SNAPSHOT`
2. **If not found locally, search GitHub** — Search `lutece-platform` and `lutece-secteur-public` orgs
3. **Find the v8 branch** — Priority: `develop_core8` > `develop8` > `develop8.x` > `develop`
4. **Verify v8 compatibility** — Fetch the remote `pom.xml` and check parent version is `8.0.0-SNAPSHOT`
5. **Read the v8 version** — Extract `<version>` from the dependency's v8 pom.xml. This is the version to use in Phase 1
6. **Clone into `~/.lutece-references/`** for later exploration

### If a dependency has NO v8 version

**STOP.** Do not proceed to Phase 1. Inform the user:

> Dependency `{artifactId}` has no Lutece 8 version. It must be migrated to v8 first before this plugin can be migrated.

Ask the user how to proceed (skip dependency, migrate it first, or provide a local path).

**Output:** A migration impact report listing everything that needs to change, **plus a dependency version map** (artifactId → v8 version) for all Lutece dependencies.

## Verification (MANDATORY before next phase)

1. Verify the impact report covers ALL categories (imports, beans, context XMLs, caches, events, REST, templates)
2. Verify ALL Lutece dependencies have a confirmed v8 version — **if any dependency is missing a v8 version, Phase 1 is BLOCKED**
3. Mark task as completed ONLY when the report is complete and all dependencies are resolved
