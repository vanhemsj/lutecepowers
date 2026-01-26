---
name: lutece-patterns
description: "Lutece 8 architecture patterns and conventions. MUST READ before writing any new Lutece code (CRUD, bean, service, DAO, XPage, daemon, templates) or answering questions about Lutece 8 architecture, layered design, or coding conventions."
---

# Lutece 8 — Architecture Patterns

Reference patterns extracted from `~/.lutece-references/lutece-core/`. Use these as the canonical way to write Lutece 8 code.

## Layered Architecture

Every Lutece feature follows 5 layers, top to bottom. Never skip a layer.

```
JspBean / XPage          ← web layer (request handling, validation, templates)
    ↓
Service                  ← business logic, cross-cutting concerns
    ↓
Home                     ← static facade (CDI lookup of DAO, cache coordination)
    ↓
DAO                      ← data access (DAOUtil, SQL, try-with-resources)
    ↓
Entity                   ← POJO (fields, getters/setters, interfaces)
```

## 1. Entity

```java
public class Task implements Serializable {
    private static final long serialVersionUID = 1L;

    private int _nIdTask;
    private String _strTitle;
    private boolean _bCompleted;
    private Timestamp _dateCreation;

    // Getters/setters only. No logic. No annotations except validation.
}
```

**Field prefixes**: `_str` (String), `_n` (int), `_b` (boolean), `_date` (Timestamp), `_list` (Collection).

**Interfaces to implement when needed**:
- `RBACResource` → RBAC permissions (`getResourceTypeCode()`, `getResourceId()`)
- `AdminWorkgroupResource` → workgroup filtering
- `IExtendableResource` → resource extension system

## 2. DAO

```java
@ApplicationScoped
public final class TaskDAO implements ITaskDAO {
    private static final String SQL_QUERY_SELECTALL = "SELECT id_task, title, completed FROM myplugin_task";
    private static final String SQL_QUERY_SELECT     = SQL_QUERY_SELECTALL + " WHERE id_task = ?";
    private static final String SQL_QUERY_INSERT     = "INSERT INTO myplugin_task ( title, completed ) VALUES ( ?, ? )";
    private static final String SQL_QUERY_UPDATE     = "UPDATE myplugin_task SET title = ?, completed = ? WHERE id_task = ?";
    private static final String SQL_QUERY_DELETE     = "DELETE FROM myplugin_task WHERE id_task = ?";

    @Override
    public void insert(Task task, Plugin plugin) {
        try (DAOUtil daoUtil = new DAOUtil(SQL_QUERY_INSERT, Statement.RETURN_GENERATED_KEYS, plugin)) {
            int nIndex = 1;
            daoUtil.setString(nIndex++, task.getTitle());
            daoUtil.setBoolean(nIndex++, task.isCompleted());
            daoUtil.executeUpdate();

            if (daoUtil.nextGeneratedKey()) {
                task.setIdTask(daoUtil.getGeneratedKeyInt(1));
            }
        }
    }

    @Override
    public Task load(int nId, Plugin plugin) {
        Task task = null;
        try (DAOUtil daoUtil = new DAOUtil(SQL_QUERY_SELECT, plugin)) {
            daoUtil.setInt(1, nId);
            daoUtil.executeQuery();

            if (daoUtil.next()) {
                task = dataToObject(daoUtil);
            }
        }
        return task;
    }

    private Task dataToObject(DAOUtil daoUtil) {
        int nIndex = 1;
        Task task = new Task();
        task.setIdTask(daoUtil.getInt(nIndex++));
        task.setTitle(daoUtil.getString(nIndex++));
        task.setCompleted(daoUtil.getBoolean(nIndex++));
        return task;
    }
}
```

**Rules**: `@ApplicationScoped`. Always try-with-resources. `nIndex++` for parameter binding. Extract `dataToObject()` to avoid duplication between `load()` and `selectAll()`.

## 3. Home (Static Facade)

```java
public final class TaskHome {
    private static ITaskDAO _dao = CDI.current().select(ITaskDAO.class).get();
    private static Plugin _plugin = PluginService.getPlugin("myplugin");

    private TaskHome() {}

    public static Task create(Task task) {
        _dao.insert(task, _plugin);
        return task;
    }

    public static Task findByPrimaryKey(int nId) {
        return _dao.load(nId, _plugin);
    }

    public static void update(Task task) {
        _dao.store(task, _plugin);
    }

    public static void remove(int nId) {
        _dao.delete(nId, _plugin);
    }

    public static List<Task> findAll() {
        return _dao.selectAll(_plugin);
    }
}
```

**Rules**: Private constructor. All methods static. CDI lookup for DAO. Plugin reference via `PluginService.getPlugin()`.

## 4. JspBean — CRUD Lifecycle

The bean is `@SessionScoped @Named` and extends `AdminFeaturesPageJspBean`.

**Method naming convention — strict**:

| Method | Role | HTTP | Returns |
|---|---|---|---|
| `getManageTasks()` | List view | GET | HTML (template) |
| `getCreateTask()` | Create form | GET | HTML (template) |
| `doCreateTask()` | Create action | POST | Redirect URL |
| `getModifyTask()` | Edit form | GET | HTML (template) |
| `doModifyTask()` | Edit action | POST | Redirect URL |
| `getConfirmRemoveTask()` | Confirmation dialog | GET | AdminMessage URL |
| `doRemoveTask()` | Delete action | POST | Redirect URL |

**Action method structure** (every `do*` follows this exact order):

```java
public String doCreateTask(HttpServletRequest request) throws AccessDeniedException {
    // 1. CSRF token validation
    if (!getSecurityTokenService().validate(request, ACTION_CREATE_TASK)) {
        throw new AccessDeniedException(ERROR_INVALID_TOKEN);
    }

    // 2. Populate bean from request
    Task task = new Task();
    populate(task, request);

    // 3. Validate (Jakarta Bean Validation)
    Set<ConstraintViolation<Task>> errors = validate(task);
    if (!errors.isEmpty()) {
        return redirect(request, VIEW_CREATE_TASK);
    }

    // 4. Business logic
    TaskHome.create(task);

    // 5. Redirect to list
    return redirectView(request, VIEW_MANAGE_TASKS);
}
```

**View method structure** (every `get*` for forms):

```java
public String getCreateTask(HttpServletRequest request) {
    Map<String, Object> model = getModel();
    model.put(MARK_TASK, new Task());
    model.put(SecurityTokenService.MARK_TOKEN,
        getSecurityTokenService().getToken(request, ACTION_CREATE_TASK));

    return getPage(PROPERTY_PAGE_TITLE_CREATE_TASK,
        TEMPLATE_CREATE_TASK, model);
}
```

## 5. Pagination & Sort (List views)

```java
public String getManageTasks(HttpServletRequest request) {
    _strCurrentPageIndex = AbstractPaginator.getPageIndex(
        request, AbstractPaginator.PARAMETER_PAGE_INDEX, _strCurrentPageIndex);
    _nItemsPerPage = AbstractPaginator.getItemsPerPage(
        request, AbstractPaginator.PARAMETER_ITEMS_PER_PAGE,
        _nItemsPerPage, _nDefaultItemsPerPage);

    List<Task> listTasks = TaskHome.findAll();

    // Sort
    String strSort = request.getParameter(Parameters.SORTED_ATTRIBUTE_NAME);
    String strAsc = request.getParameter(Parameters.SORTED_ASC);
    if (strSort != null) {
        Collections.sort(listTasks, new AttributeComparator(strSort, Boolean.parseBoolean(strAsc)));
    }

    // Paginate
    UrlItem url = new UrlItem(JSP_MANAGE_TASKS);
    if (strSort != null) {
        url.addParameter(Parameters.SORTED_ATTRIBUTE_NAME, strSort);
        url.addParameter(Parameters.SORTED_ASC, strAsc);
    }

    LocalizedPaginator<Task> paginator = new LocalizedPaginator<>(
        listTasks, _nItemsPerPage, url.getUrl(),
        AbstractPaginator.PARAMETER_PAGE_INDEX, _strCurrentPageIndex, getLocale());

    Map<String, Object> model = getModel();
    model.put(MARK_PAGINATOR, paginator);
    model.put(MARK_TASK_LIST, paginator.getPageItems());

    return getPage(PROPERTY_PAGE_TITLE_MANAGE, TEMPLATE_MANAGE_TASKS, model);
}
```

## 6. XPage (Front-office)

```java
@RequestScoped
@Named("myplugin.xpage.tasks")
public class TasksApp extends AbstractXPageApplication {
    @Inject
    private TaskService _taskService;

    @Override
    public XPage getPage(HttpServletRequest request, int nMode, Plugin plugin) {
        XPage page = new XPage();
        page.setTitle(I18nService.getLocalizedString(PROPERTY_PAGE_TITLE, request.getLocale()));
        page.setContent(getTaskList(request));
        return page;
    }
}
```

**Rules**: `@RequestScoped`. Implements `XPageApplication`. Declared in plugin.xml `<applications>`.

## 7. Daemon (Background task)

```java
public class TaskCleanupDaemon extends Daemon {
    @Override
    public void run() {
        int nCleaned = TaskHome.removeExpired();
        setLastRunLogs("Cleaned " + nCleaned + " expired tasks");
    }
}
```

Declared in plugin.xml:
```xml
<daemons>
    <daemon>
        <daemon-id>taskCleanup</daemon-id>
        <daemon-name>myplugin.daemon.taskCleanup.name</daemon-name>
        <daemon-description>myplugin.daemon.taskCleanup.description</daemon-description>
        <daemon-class>fr.paris.lutece.plugins.myplugin.daemon.TaskCleanupDaemon</daemon-class>
    </daemon>
</daemons>
```

Configuration via properties: `daemon.taskCleanup.interval=3600`, `daemon.taskCleanup.onstartup=0`.
Signal on demand: `AppDaemonService.signalDaemon("taskCleanup")`.

## 8. CDI Patterns — Quick Reference

| Need | Pattern |
|---|---|
| Singleton service | `@ApplicationScoped` on class |
| Per-request bean | `@RequestScoped` on class |
| Session bean (admin) | `@SessionScoped @Named` on class |
| Field injection | `@Inject private MyService _service;` |
| Static lookup (Home) | `CDI.current().select(IMyDAO.class).get()` |
| Multiple implementations | `CDI.current().select(IProvider.class)` → `.stream().filter(...)` |
| Fire event | `CDI.current().getBeanManager().getEvent().fire(new MyEvent(...))` |
| Observe event | `public void onEvent(@Observes MyEvent event) { }` |
| Config property | `@Inject @ConfigProperty(name = "my.key", defaultValue = "x")` |

## 9. Configuration Access

```java
// Properties (static, from .properties files)
String val = AppPropertiesService.getProperty("myplugin.my.key");
int n = AppPropertiesService.getPropertyInt("myplugin.items.per.page", 50);

// Datastore (runtime, from database — overrides properties)
String ds = DatastoreService.getInstanceDataValue("myplugin.setting", "default");
DatastoreService.setInstanceDataValue("myplugin.setting", "newValue");

// In Freemarker templates
// #{dskey{myplugin.setting}}
```

## 10. Security Checklist

Every admin feature MUST:
1. Call `init(request, RIGHT_MANAGE_TASKS)` to check the user right
2. Include `SecurityTokenService.MARK_TOKEN` in every form model
3. Validate `getSecurityTokenService().validate(request, ACTION)` in every `do*` method
4. Filter collections with `RBACService.getAuthorizedCollection()` when RBAC is enabled
5. Filter by workgroup with `AdminWorkgroupService.getAuthorizedCollection()` when needed
