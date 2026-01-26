# Lutecepowers-v8

Claude Code plugin for **Lutece 8** framework development.

## Installation

```bash
/plugin marketplace add vanhemsj/lutecepowers
/plugin install lutecepowers-v8
```

## Hook (SessionStart)

At session start, the plugin runs 3 hooks automatically:

1. **System prompt injection** — Informs Claude of available skills and instructs it to always consult the reference repos (`~/.lutece-references/`) via Read/Grep/Glob before writing any Lutece code
2. **Reference repos clone/update** — Clones 15 v8 repositories into `~/.lutece-references/` on first run, then pulls **all** repos in that directory at every session start (including repos added manually). This ensures references are always up to date.
3. **Rules copy** — Runs `scripts/lutece-rules-setup.sh` to copy rule templates into the target project's `.claude/rules/`

## Rules

Rules are constraints automatically loaded when Claude touches matching files. They are copied to the target project's `.claude/rules/` at session start.

| Rule | Scope | Description |
|------|-------|-------------|
| `java-conventions` | `**/*.java` | Global Java constraints: jakarta imports (with JDK javax exceptions), beans.xml requirement, field naming prefixes |
| `web-bean` | `**/web/**/*.java` | JspBean/XPage constraints: CDI annotations, CRUD lifecycle, security tokens, pagination |
| `service-layer` | `**/service/**/*.java` | Service layer constraints: CDI scopes, injection, events, configuration, cache integration |
| `dao-patterns` | `**/business/**/*.java` | DAOUtil lifecycle, SQL constants, Home facade, CDI lookup |
| `template-back-office` | `**/templates/admin/**/*.html` | Freemarker macros, layout structure, i18n, null safety, BS5/Tabler already loaded (prefer macros), vanilla JS, no CDN |
| `template-front-office` | `**/templates/skin/**/*.html` | BS5/Tabler/core JS already loaded (no imports), BS5 classes only, vanilla JS (no jQuery), no CDN |
| `jsp-admin` | `**/*.jsp` | Admin feature JSP boilerplate, bean naming, errorPage |
| `plugin-descriptor` | `**/plugins/*.xml` | Mandatory tags, icon-url, core-version-dependency |
| `messages-properties` | global | i18n constraints: no prefix in .properties, prefix in Java/templates |
| `drykissyagni` | global | DRY, KISS, YAGNI principles |
| `dependency-references` | global | Lutece deps: auto-fetch sources into `~/.lutece-references/`, v8 branch detection, warn if no v8. External deps: use Context7 MCP if available, suggest install otherwise |

## Skills

Skills are multi-step procedures loaded progressively by Claude on demand.

| Skill | Description |
|-------|-------------|
| `lutece-migration-v8` | Migration v7 → v8 (Spring → CDI/Jakarta). 15 phases orchestrated via Claude Code Tasks (`TaskCreate`/`TaskUpdate` with `blockedBy` dependencies). |
| `lutece-scaffold` | Interactive plugin scaffold generator. Creates plugins with optional XPage, Cache, RBAC and Site features. |
| `lutece-site` | Interactive site generator. Creates a site with database configuration and optional plugin dependencies. |
| `lutece-workflow` | Rules and patterns for creating/modifying workflow modules. Tasks, CDI producers, components, templates. |
| `lutece-dao` | DAO and Home layer patterns: DAOUtil lifecycle, SQL constants, Home static facade, CDI lookup. |
| `lutece-cache` | Cache implementation: AbstractCacheableService, CDI initialization, invalidation via CDI events. |
| `lutece-rbac` | RBAC implementation: entity permissions, ResourceIdService, plugin.xml declaration, JspBean authorization. |
| `lutece-lucene-indexer` | Plugin-internal Lucene search: custom index, daemon, CDI events, batch processing. |
| `lutece-patterns` | Lutece 8 architecture reference: layered architecture, CDI patterns, CRUD lifecycle, pagination, XPages, daemons, security checklist. |
| `using-lutecepowers` | Introduction to the lutecepowers skills library. |

## Migration Samples

The `migrations-samples/` directory contains detailed analyses of real v7 → v8 migrations (14 plugins/modules/libraries). Each file documents the exact diffs per category (POM, imports, CDI, events, cache, templates, tests, etc.). These samples are consulted by the migration skill and by the v8-reviewer agent as reference.

## Agents

Agents are specialized subagents that run in their own context window with restricted tools.

| Agent | Model | Tools | Description |
|-------|-------|-------|-------------|
| `lutece-v8-reviewer` | Haiku | Read, Grep, Glob, Bash | Reviews a Lutece plugin for v8 compliance. Checks CDI annotations, jakarta imports, POM dependencies, beans.xml, Spring residues. Produces a structured PASS/WARN/FAIL report. |

## Tests

Tests are run via the Claude Agent SDK and defined in `tests/tests.json`. Each test case spins up a Claude session with the plugin loaded against a fixture project, then asserts expected behavior (tool usage, file reads, agent delegation).

```bash
# Install dependencies (first time only)
pip install -r tests/requirements.txt

# Run all tests
python3 -m pytest tests/test.py -v

# Run a specific test
python3 -m pytest tests/test.py -v -k "v8_reviewer"

# Run tests in parallel
python3 -m pytest tests/test.py -v -n 4
```

Latest results: [TEST_REPORT.md](TEST_REPORT.md)

## Manual Testing

To test the plugin locally against a Lutece project:

```bash
cd /path/to/your-lutece-plugin-to-test
claude --plugin-dir /path/to/lutecepowers-v8
```

This loads the plugin without installing it from the marketplace. SessionStart hooks will run normally (reference repos clone, rules copy, system prompt injection).
