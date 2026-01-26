---
name: using-lutecepowers
description: "Introduction to lutecepowers skills library for Lutece framework development"
---

# Using Lutecepowers

Lutecepowers is a skills library for developing with the **Lutece framework**. It provides structured guides for common tasks.

## Architecture

Lutecepowers uses two complementary mechanisms:

**Rules** (`.claude/rules/`) — concise constraints, always loaded, path-scoped:
- `java-conventions.md` (`**/*.java`) — CDI annotations, Jakarta imports, naming
- `freemarker.md` (`**/*.html`) — layout macros, JSP paths, reference sources
- `dao-patterns.md` (`**/business/**/*.java`) — DAOUtil, SQL constants, Home facade
- `messages-properties.md` (global) — i18n golden rule, key naming
- `jsp-admin.md` (`**/*.jsp`) — JSP boilerplate, bean naming
- `plugin-descriptor.md` (`**/plugins/*.xml`) — plugin.xml mandatory tags

**Skills** — full reference with code templates, loaded on demand:

| Skill | Use Case |
|-------|----------|
| `lutece-dao` | Full DAO/Home code templates and patterns |
| `lutece-freemarker` | Full list/form page templates and macro catalog |
| `lutece-workflow` | Creating and modifying workflow modules |
| `lutece-rbac` | Implementing RBAC in a plugin |
| `lutece-cache` | Implementing cache (AbstractCacheableService, CDI events) |
| `lutece-lucene-indexer` | Plugin-internal Lucene search (custom index, daemon, batch) |
| `lutece-scaffold` | Interactive plugin scaffold generator |
| `lutece-site` | Interactive site generator |
| `lutece-migration-v8` | Migrating a plugin from v7 (Spring) to v8 (CDI/Jakarta) |

## How to Use

Rules apply automatically when you touch relevant files. Skills are loaded on demand:

```
Use the lutece-dao skill for full DAO code examples
Use the lutece-workflow skill to create a new workflow task
Use the lutece-migration-v8 skill to create a migration plan for plugin-example
```

## Reference Sources

Skills rely on **reference repositories** cloned locally to `~/.lutece-references/`. These sources are real Lutece 8 implementations and MUST be consulted before any code creation or modification.

**Setup:** Automatic at Claude Code startup via the SessionStart hook.

Available repos:
- `lutece-core` — Core framework (DAO, Home, Service, JSPBean, MVC patterns)
- `lutece-form-plugin-forms` — Forms plugin
- `lutece-genattrs-plugin-genericattributes` — Generic attributes
- `lutece-wf-plugin-workflow` — Workflow engine
- `lutece-wf-library-workflow-core` — Workflow interfaces (ITask, ITaskType...)
- `lutece-wf-module-workflow-forms` — Reference workflow module
- `lutece-wf-module-workflow-forms-automatic-assignment` — Automatic assignment
- `lutece-wf-module-workflow-upload` — Upload in workflow
- `lutece-wf-module-workflow-formstopdf` — PDF generation
- `lutece-tech-plugin-rest` — REST API
- `lutece-tech-plugin-asynchronousupload` — Asynchronous upload
- `lutece-tech-plugin-filegenerator` — File generation
- `lutece-tech-library-httpaccess` — HTTP client
- `lutece-tech-library-signrequest` — Request signing

## Adding Skills

To add a new skill:

1. Create `skills/{skill-name}/SKILL.md`
2. Add YAML frontmatter with `name` and `description`
3. Write the skill content in markdown

## Lutece Framework

Lutece is a Java-based CMS/portal framework. Key concepts:

- **Plugins**: Modular extensions
- **JSPBeans**: Admin controllers (back-office)
- **XPages**: Front-office applications
- **DAOs/Homes**: Data access layer
- **Services**: Business logic layer
- **Workflow**: State machine for business processes

## Version 8 Changes

Lutece v8 introduces:

- **CDI** instead of Spring for dependency injection
- **Jakarta EE** namespaces (`jakarta.*` instead of `javax.*`)
- **Annotations** instead of XML configuration
