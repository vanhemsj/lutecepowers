---
name: lutece-scaffold
description: "Interactive Lutece 8 plugin scaffold generator. Creates plugins with optional XPage, Cache, RBAC and Site features. Supports parent-child entity relationships."
---

# Lutece 8 Plugin Scaffold Generator

Generate complete Lutece 8 plugin scaffolds through interactive questioning.

## Process Overview

```
Phase 1: Plugin Info     → Name, description, output directory
Phase 2: Entities        → Entities with fields, i18n labels (en/fr) and parent-child relations
Phase 3: Features        → XPage, Cache, RBAC, Site, Workflow (optional)
Phase 4: Generation      → Run the script, generate the scaffold
Phase 5: Feature Planning → Plan features (rules auto-loaded from .claude/rules/)
```

## Phase 1: Plugin Info

Ask one question at a time:
1. Plugin name? (lowercase, no spaces, e.g., "taskmanager")
2. Plugin description? (en/fr)
3. Output directory? (where to generate)

## Phase 2: Entities

For each entity:
1. Entity name? (PascalCase, e.g., "Project")
2. Entity label? (en/fr for i18n)
3. **Parent entity?** (optional - parent entity name, e.g., "Project" for Task)
4. Table name? (default: pluginname_entityname)
5. Fields? (one by one or batch)

For each field:
- Field name? (camelCase)
- Field type? (string, int, long, boolean, double, timestamp, date, etc.)
- Field label? (en/fr for i18n)
- Required? (yes/no)

After each entity: "Add another entity?"

### Sub-Entity Support (Parent-Child Relations)

When an entity has a `parentEntity`, the scaffold automatically generates:

**Business Layer:**
- FK field `idParent` in the child entity (e.g., `idProject` for Task)
- `IEntityDAO`: `selectByParentId()`, `deleteByParentId()`
- `EntityDAO`: Implementation with SQL queries
- `EntityHome`: `findByParentId()`, `removeByParentId()`

**Service Layer (like forms plugin):**
- `EntityService`: Orchestrates cascade deletes
- `@Transactional` support for data integrity
- Recursive cascade through entity hierarchy

**Web Layer:**
- JSPBean: Filters list by parent, passes parent context
- Templates: Breadcrumb navigation, links to children
- i18n: Labels for child entity buttons

**Example Hierarchy:**
```
Project (parent)
└── Task (parentEntity: "Project")
    └── Comment (parentEntity: "Task")
```

When deleting a Project:
1. ProjectService.remove() is called
2. It calls TaskService.remove() for each Task
3. TaskService.remove() calls CommentService.remove() for each Comment
4. Then deletes the Task
5. Then deletes the Project

## Phase 3: Optional Features

### XPage (Front-office)
- "Do you want an XPage (front-office page)?"
- If yes: XPage name? Title (en/fr)?

### Cache
- "Do you want a cache system?"

### RBAC (Permissions)
- "Do you want RBAC permissions?"
- If yes: Which permissions? (CREATE, MODIFY, DELETE, VIEW, etc.)

### Site (Test environment)
- "Do you want to generate a test site?"
- If yes: **Display default credentials table and ask for confirmation:**

```
Database configuration for the test site:

| Parameter | Value |
|-----------|-------|
| Host      | localhost |
| Port      | 3306 |
| Database  | lutece_{pluginName} |
| User      | root |
| Password  | ??? |

Are these values correct? (yes/no or specify changes)
```

**IMPORTANT:**
- ALWAYS display this table BEFORE generating the site
- WAIT for explicit user confirmation
- If the user says "no" or specifies changes, update the values
- NEVER generate the site without credentials confirmation

### Workflow (Workflow integration)
- "Do you want to integrate the plugin with the workflow system?"

The workflow module generates:
- A separate `module-workflow-{pluginName}` module
- For each entity: a state change task (TaskXxxStateChange)
- CDI configuration with producers (@Produces ITaskType, ITaskConfigService)
- Configuration, form and information templates
- FR/EN i18n for tasks

**Generated structure:**
```
module-workflow-{pluginName}/
├── src/java/.../workflow/modules/{pluginName}/
│   ├── business/
│   │   ├── Task{Entity}StateChangeConfig.java
│   │   └── Task{Entity}StateChangeConfigDAO.java
│   ├── service/
│   │   ├── Task{Entity}StateChange.java (extends SimpleTask)
│   │   ├── TaskType{Entity}StateChangeProducer.java (@Produces ITaskType)
│   │   └── Task{Entity}StateChangeConfigServiceProducer.java
│   └── web/
│       └── {Entity}StateChangeTaskComponent.java
├── webapp/WEB-INF/
│   ├── conf/plugins/workflow-{pluginName}.properties
│   ├── plugins/workflow-{pluginName}.xml
│   └── templates/admin/plugins/workflow/modules/{pluginName}/
│       ├── task_{entity}_statechange_config.html
│       ├── task_{entity}_statechange_form.html
│       └── task_{entity}_statechange_information.html
└── src/sql/plugins/workflow/modules/create_db_workflow-{pluginName}.sql
```

## Phase 4: Generation

1. Build JSON config from answers
2. Write config to `/tmp/{pluginName}-config.json`
3. Run: `bash "${CLAUDE_PLUGIN_ROOT}/scripts/lutece-scaffold.sh" <config> <output>`
4. Show summary of generated files

### JSON Config Format

```json
{
  "pluginName": "taskmanager",
  "pluginDescription": {
    "en": "Task management plugin",
    "fr": "Plugin de gestion des tâches"
  },
  "packageBase": "fr.paris.lutece.plugins.taskmanager",
  "features": {
    "xpage": { "enabled": true, "name": "mytasks", "title": { "en": "My Tasks", "fr": "Mes tâches" } },
    "cache": { "enabled": true },
    "rbac": { "enabled": true, "permissions": ["CREATE", "MODIFY", "DELETE", "VIEW"] },
    "site": { "enabled": true, "name": "site-taskmanager", "database": { "host": "localhost", "port": 3306, "name": "lutece_taskmanager", "user": "root", "password": "<CONFIRM WITH USER>" } },
    "workflow": { "enabled": true }
  },
  "entities": [
    {
      "name": "Project",
      "label": { "en": "Project", "fr": "Projet" },
      "tableName": "taskmanager_project",
      "fields": [
        { "name": "name", "type": "string", "required": true, "label": { "en": "Name", "fr": "Nom" } },
        { "name": "description", "type": "longtext", "required": false, "label": { "en": "Description", "fr": "Description" } }
      ]
    },
    {
      "name": "Task",
      "label": { "en": "Task", "fr": "Tâche" },
      "tableName": "taskmanager_task",
      "parentEntity": "Project",
      "fields": [
        { "name": "title", "type": "string", "required": true, "label": { "en": "Title", "fr": "Titre" } },
        { "name": "completed", "type": "boolean", "required": true, "label": { "en": "Completed", "fr": "Terminée" } }
      ]
    }
  ]
}
```

### Field Types

| Type | Java | SQL |
|------|------|-----|
| string | String | VARCHAR(255) |
| longtext | String | LONG VARCHAR |
| int, number | int | INT |
| long | long | BIGINT |
| boolean | boolean | SMALLINT |
| double | double | DOUBLE |
| timestamp | Timestamp | TIMESTAMP |
| date | Date | DATE |

### Generated Files

**Base:**
- `plugin-{name}/pom.xml`, `beans.xml`
- `{Entity}.java`, `I{Entity}DAO.java`, `{Entity}DAO.java`, `{Entity}Home.java`
- `{Entity}Service.java` - Service layer with cascade support
- `{Entity}JspBean.java` (with `@SessionScoped`, `@Named`)
- `manage_{entity}s.html`, `create_{entity}.html`, `modify_{entity}.html`
- `Manage{Entity}s.jsp`
- `{plugin}_messages.properties`, `_en.properties`, `_fr.properties`
- `create_db_{plugin}.sql`, `{plugin}.xml`

**For Child Entities:**
- FK field `idParent` auto-generated
- Index on FK column in SQL
- `findByParentId()`, `removeByParentId()` methods
- Breadcrumb navigation in templates
- Parent context in JSPBean

**If XPage:** `{Plugin}XPage.java`, `list_{entity}s.html` (skin)

**If Cache:** `service/cache/{Plugin}CacheService.java`

**If RBAC:** `service/{Plugin}ResourceIdService.java`, `service/{Plugin}Plugin.java`

**If Site:** `sites/{site-name}/` with DB config

**If Workflow:** `module-workflow-{plugin}/` with Task, Producer, Component, Templates, SQL

## Example Flow

```
User: /lutece-scaffold
Assistant: Plugin name?
User: taskmanager
Assistant: Description? (en/fr)
User: Task manager / Gestionnaire de tâches
Assistant: Output directory?
User: /home/user/plugins
Assistant: First entity name?
User: Project
Assistant: Label? (en/fr)
User: Project / Projet
Assistant: Parent entity? (leave empty if none)
User: (empty)
Assistant: Fields?
User: name:string:Nom, description:longtext:Description
Assistant: Another entity?
User: yes, Task
Assistant: Label? (en/fr)
User: Task / Tâche
Assistant: Parent entity?
User: Project
Assistant: Fields?
User: title:string:Titre, completed:boolean:Terminée
Assistant: Another entity?
User: no
Assistant: XPage?
User: yes, mytasks / Mes tâches
Assistant: Cache?
User: yes
Assistant: RBAC?
User: yes, CREATE, MODIFY, DELETE, VIEW
Assistant: Test site?
User: yes
Assistant:
Database configuration for the test site:

| Parameter | Value |
|-----------|-------|
| Host      | localhost |
| Port      | 3306 |
| Database  | lutece_taskmanager |
| User      | root |
| Password  | ??? |

Are these values correct?
User: yes, password = root
Assistant: Workflow integration?
User: yes

Scaffold generated!
- Plugin: plugin-taskmanager/
- Entity: Project (2 fields)
- Entity: Task (2 fields) → child of Project
- XPage: mytasks
- Cache: TaskmanagerCacheService
- RBAC: 4 permissions
- Site: site-taskmanager
- Workflow: module-workflow-taskmanager/

Next steps:
1. cd plugin-taskmanager && mvn clean install
2. cd module-workflow-taskmanager && mvn clean install
3. cd sites/site-taskmanager && mvn lutece:site-assembly
4. cd target/site-taskmanager-1.0.0-SNAPSHOT/WEB-INF/sql && ant
5. cd ../../../.. && mvn liberty:dev

What specific features do you want to develop?
```

## Phase 5: Feature Planning

After scaffold generation, ask: **"What specific features do you want to develop?"**

**IMPORTANT:** Before developing any feature:
1. **Search `~/.lutece-references/`** for real implementation patterns (Read/Grep/Glob on the reference repos)
3. Write a plan in `docs/plans/YYYY-MM-DD-<feature>.md`
4. Validate the plan BEFORE coding

## Scaffold Limitations

**IMPORTANT:** The scaffold has a defined scope. When the user requests something outside this scope, you MUST clearly inform them and offer Phase 5 (post-scaffold development) as an alternative.

### Features supported by the scaffold

| Feature | Supported | Details |
|---------|-----------|---------|
| Admin CRUD (JSPBean) | Yes | For all entities |
| XPage (front-office) | Yes | **First entity only** (list view) |
| Cache | Yes | Shared CacheService for all entities |
| RBAC | Yes | Permissions on the first entity |
| Test site | Yes | With DB config |
| Workflow | Yes | Separate module, tasks for root entities |
| Parent-child relations (1-N) | Yes | Cascade delete via Service |

### What the scaffold does NOT support

If the user requests any of these features, **clearly inform them** that the scaffold does not generate it, then offer to develop it in Phase 5:

| Request | Response to give |
|---------|-----------------|
| REST API | "The scaffold does not generate a REST API. I can develop it in Phase 5 after generation." |
| Many-to-many relations | "The scaffold only supports parent-child relations (1-N). For N-N, I will create the junction table manually in Phase 5." |
| Search / Lucene | "Lucene indexing is not generated. Use `/lutece-lucene-indexer` after the scaffold." |
| Unit tests | "The scaffold creates the test structure but not the test files. I can write them in Phase 5." |
| Bean Validation (JSR-303) | "Server-side validation is not generated. To be added manually in Phase 5." |
| Import/Export (CSV, batch) | "Not supported by the scaffold. Phase 5." |
| Multiple admin features | "The scaffold generates a single admin feature pointing to the first entity. Other entities are accessible via parent-child navigation or Phase 5." |
| Enum fields | "The `enum` type does not exist in the scaffold. Use `int` or `string` and handle the enum in Phase 5." |
| Computed / virtual fields | "Not supported. To be implemented in the Service in Phase 5." |
| Multiple file upload | "The scaffold handles one `file` field (type `File`) per entity but not multi-upload. Phase 5." |

### Supported field types

Recognized types are: `string`, `longtext`, `int`, `long`, `boolean`, `double`, `float`, `timestamp`, `date`, `time`, `decimal`, `file`.

**WARNING — Silent fallback:** Any unrecognized type (e.g., `email`, `url`, `json`, `enum`, `uuid`) will be silently converted to `String / VARCHAR(255)`. You MUST warn the user if the requested type is not in the list above:

> "The type `{type}` is not recognized by the scaffold and will be treated as `string` (VARCHAR 255). If you need specific behavior (validation, format), we will add it in Phase 5."

### XPage: first entity limitation

The generated XPage only covers the first entity of the plugin (list view with pagination). If the user wants front-office pages for other entities, inform them:

> "The XPage is generated only for the `{firstEntity}` entity. For other entities, I will develop additional views in Phase 5."

## Key Principles

- **One question at a time** - Do not overwhelm
- **Accept shortcuts** - "title:string:Title" = field with type and EN label
- **Smart defaults** - Minimize input
- **Inform on limits** - Always warn if a request falls outside scaffold scope
- **i18n from the start** - Always ask for en/fr
- **Parent-child support** - Ask for parent entity for each entity
- **Service layer** - Cascade is handled at the service level, not SQL
- **Rules auto-loaded** - `.claude/rules/` constraints apply automatically when touching relevant files
