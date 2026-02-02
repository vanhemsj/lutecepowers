# Phase 3: Spring to CDI Migration

## 3.1 Remove Spring Context XML Files

1. Delete all `*_context.xml` files (e.g., `webapp/WEB-INF/conf/plugins/myPlugin_context.xml`)
2. Before deleting, **catalog every bean** defined in these files — each one must be migrated

## 3.2 Add CDI beans.xml

Create `src/main/resources/META-INF/beans.xml`:
```xml
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/beans_4_0.xsd"
       version="4.0" bean-discovery-mode="annotated">
</beans>
```

## 3.3 Annotate DAO Classes

Every DAO class must get `@ApplicationScoped`:
```java
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class MyDAO implements IMyDAO { ... }
```

## 3.4 Annotate Service Classes

Every service class must get `@ApplicationScoped`:
```java
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class MyService { ... }
```

- Remove `final` keyword from class if present (CDI cannot proxy final classes)
- Remove private constructors used for singleton enforcement
- Remove static `_instance` / `_singleton` fields
- Convert `getInstance()` to use CDI:
```java
public static MyService getInstance() {
    return CDI.current().select(MyService.class).get();
}
```

## 3.5 Replace SpringContextService Calls

| v7 Pattern | v8 Pattern |
|-----------|-----------|
| `SpringContextService.getBean("beanId")` | `CDI.current().select(InterfaceType.class).get()` |
| `SpringContextService.getBean("namedBean")` | `CDI.current().select(Type.class, NamedLiteral.of("namedBean")).get()` |
| `SpringContextService.getBeansOfType(Type.class)` | `CDI.current().select(Type.class)` (returns `Instance<Type>`) |
| `SpringContextService.getBeansOfType(Type.class)` in loop | `CDI.current().select(Type.class).forEach(...)` |

## 3.6 Replace Static DAO References in Home Classes

- Remove `final` keyword from Home classes (v8 convention: no `final` on DAO, Service, or Home classes)

```java
// BEFORE
public final class MyHome {
    private static IMyDAO _dao = SpringContextService.getBean("myDAO");
}

// AFTER
public class MyHome {
    private static IMyDAO _dao = CDI.current().select(IMyDAO.class).get();
}
```

## 3.7 CDI Producers (for complex bean definitions)

When a Spring XML bean had constructor args, property values, or list configurations, create a Producer class.

**CRITICAL: Check the v8 API first.** Before writing a Producer, read the **v8 source** of the class being instantiated in `~/.lutece-references/`. The class constructors and setters may have changed in v8 (e.g., constructor args removed because the class is now CDI-managed with `@Inject`). If the v8 class is `@ApplicationScoped`, you probably don't need a Producer at all — just `@Inject` it directly.

```java
@ApplicationScoped
public class MyProducer {

    @Produces
    @Named("beanName")
    @ApplicationScoped
    public MyType produce() {
        // Recreate the bean configuration from the old XML
        // BUT use the v8 constructor/API, not the v7 one
        return new MyType(...);
    }
}
```

## 3.8 CDI Injection

For CDI-managed beans (classes annotated with `@ApplicationScoped`), prefer `@Inject` over `CDI.current().select()`:

```java
@Inject
private IMyDAO _dao;

@Inject
@Named("namedBean")
private IMyService _service;
```

Use `Instance<T>` for optional or multiple beans:
```java
@Inject
Instance<IMyProvider> _providers;

// Optional bean
@Inject
@Named("optionalBean")
Instance<IMyProvider> _optionalProvider;
```

## XPage Migration

XPage classes require special CDI scope annotations:

### Pattern: XPage with session state → `@SessionScoped`
```java
// BEFORE (v7)
@Controller( xpageName = "myXPage", pageTitleI18nKey = "...", pagePathI18nKey = "..." )
public class MyXPage extends MVCApplication {
    private static MyService _service = SpringContextService.getBean( MyService.BEAN_NAME );
    private ICaptchaSecurityService _captchaSecurityService = new CaptchaSecurityService();
}

// AFTER (v8)
@SessionScoped
@Named( "myplugin.xpage.myXPage" )
@Controller( xpageName = "myXPage", pageTitleI18nKey = "...", pagePathI18nKey = "...", securityTokenEnabled=false )
public class MyXPage extends MVCApplication {
    @Inject
    private MyService _service;
    @Inject
    @Named(BeanUtils.BEAN_CAPTCHA_SERVICE)
    private Instance<ICaptchaService> _captchaService;
    @Inject
    private SecurityTokenService _securityTokenService;
}
```

### Pattern: XPage without session state → `@RequestScoped`
```java
@RequestScoped
@Named( "myplugin.xpage.myOtherXPage" )
@Controller( xpageName = "myOtherXPage", ... )
public class MyOtherXPage extends MVCApplication { ... }
```

### Key XPage migration rules:
1. Add `@SessionScoped` or `@RequestScoped` (choose based on whether the XPage maintains state)
2. Add `@Named("pluginName.xpage.xpageName")` to identify the bean
3. Replace all `SpringContextService.getBean()` with `@Inject`
4. Replace `SecurityTokenService.getInstance()` with `@Inject private SecurityTokenService`
5. Replace `WorkflowService.getInstance()` with `@Inject private WorkflowService`
6. Replace `new CaptchaSecurityService()` with `@Inject @Named(BeanUtils.BEAN_CAPTCHA_SERVICE) Instance<ICaptchaService>`
7. Replace static upload handler access with `@Inject`

## EntryType Classes (GenericAttributes-based plugins)

EntryType classes (extending `AbstractEntryType*`) need:
1. `@ApplicationScoped`
2. `@Named("pluginName.entryTypeName")` matching the bean name from the old context XML
3. Anonymization types injected via `@Inject` method with `@Named` parameters
4. Upload handler injected via `@Inject` instead of static access

```java
@ApplicationScoped
@Named( "myplugin.entryTypeText" )
public class EntryTypeText extends AbstractEntryTypeText {
    @Inject
    private MyAsynchronousUploadHandler _uploadHandler;

    @Inject
    public void addAnonymizationTypes(
        @Named("genericattributes.entryIdAnonymizationType") IEntryAnonymizationType entryId,
        @Named("genericattributes.entryCodeAnonymizationType") IEntryAnonymizationType entryCode,
        // ... other anonymization types
    ) {
        setAnonymizationTypes( List.of( entryId, entryCode, ... ) );
    }

    @Override
    public AbstractGenAttUploadHandler getAsynchronousUploadHandler() {
        return _uploadHandler;
    }
}
```

## CDI Producers with @ConfigProperty

When Spring XML beans had property values, create a Producer that reads from `.properties` via `@ConfigProperty`:

```java
@ApplicationScoped
public class MyProducer {
    @Produces
    @ApplicationScoped
    @Named("myplugin.myBeanName")
    public MyType produce(
        @ConfigProperty(name = "myplugin.myBean.propertyA") String propA,
        @ConfigProperty(name = "myplugin.myBean.propertyB") String propB
    ) {
        return new MyType(propA, propB);
    }
}
```

The corresponding `.properties` file must contain the values:
```properties
myplugin.myBean.propertyA=valueA
myplugin.myBean.propertyB=valueB
```

## Business Objects - Serializable

Business objects used in `@SessionScoped` beans or passed through CDI events should implement `Serializable`:

```java
public class MyBusinessObject implements Cloneable, Serializable {
    private static final long serialVersionUID = 1L;
    // ...
}
```

## CDI Scope for Spring Prototype Beans

Spring `scope="prototype"` beans become CDI `@Dependent`:

```java
// v7 Spring XML: <bean id="myBean" class="..." scope="prototype" />
// v8:
@Dependent
@Named("myBean")
public class MyBean { ... }
```

## Transaction Annotation Simplification

```java
// BEFORE (v7)
@Transactional(MyPlugin.BEAN_TRANSACTION_MANAGER)

// AFTER (v8) - no transaction manager reference
@Transactional
```

Import change: `org.springframework.transaction.annotation.Transactional` → `jakarta.transaction.Transactional`

## Constructor Injection Pattern (Workflow Tasks/Components)

For workflow task components, use constructor injection:

```java
@ApplicationScoped
@Named("myModule.myTaskComponent")
public class MyTaskComponent extends NoFormTaskComponent {
    @Inject
    public MyTaskComponent(
        @Named("myModule.taskType") ITaskType taskType,
        @Named("myModule.taskConfigService") ITaskConfigService configService
    ) {
        setTaskType(taskType);
        setTaskConfigService(configService);
    }
}
```

## TaskType Producer Pattern (Workflow modules)

TaskType beans from Spring XML become CDI producers with `@ConfigProperty`:

```java
@ApplicationScoped
public class MyTaskTypeProducer {
    @Produces @ApplicationScoped
    @Named("myModule.taskType")
    public ITaskType produce(
        @ConfigProperty(name = "myModule.taskType.key") String key,
        @ConfigProperty(name = "myModule.taskType.titleI18nKey") String titleI18nKey,
        @ConfigProperty(name = "myModule.taskType.beanName") String beanName,
        @ConfigProperty(name = "myModule.taskType.configBeanName") String configBeanName,
        @ConfigProperty(name = "myModule.taskType.configRequired", defaultValue = "false") boolean configRequired,
        @ConfigProperty(name = "myModule.taskType.taskForAutomaticAction", defaultValue = "false") boolean taskForAutomaticAction
    ) {
        TaskType t = new TaskType();
        t.setKey(key); t.setTitleI18nKey(titleI18nKey); t.setBeanName(beanName);
        t.setConfigBeanName(configBeanName); t.setConfigRequired(configRequired);
        t.setTaskForAutomaticAction(taskForAutomaticAction);
        return t;
    }
}
```

With corresponding `.properties`:
```properties
myModule.taskType.key=myTaskKey
myModule.taskType.titleI18nKey=module.workflow.mymodule.task_title
myModule.taskType.beanName=myModule.myTask
myModule.taskType.configBeanName=myModule.myTaskConfig
myModule.taskType.configRequired=true
myModule.taskType.taskForAutomaticAction=false
```

## CDI Impl Classes for Abstract Library Services

When a library provides abstract service classes (e.g., `ActionService` from `library-workflow-core`), create empty CDI implementation classes:

```java
@ApplicationScoped
@Named(ActionService.BEAN_SERVICE)
public class ActionServiceImpl extends ActionService {
    // Empty - provides CDI annotations for parent abstract class
}
```

This is needed because CDI cannot proxy abstract classes without a concrete subclass.

## Default Constructor for CDI Proxies

CDI-managed classes with `@Inject` constructor MUST also have a default (no-arg) constructor:

```java
@ApplicationScoped
@Named("myModule.myTaskComponent")
public class MyTaskComponent extends NoFormTaskComponent {
    MyTaskComponent() { } // Required for CDI proxy

    @Inject
    public MyTaskComponent(
        @Named("myModule.taskType") ITaskType taskType,
        @Named("myModule.configService") ITaskConfigService configService
    ) {
        setTaskType(taskType);
        setTaskConfigService(configService);
    }
}
```

## DAOUtil try-with-resources

Replace manual `daoUtil.free()` with try-with-resources:

```java
// BEFORE (v7)
DAOUtil daoUtil = new DAOUtil(SQL_QUERY, plugin);
daoUtil.setInt(1, id);
daoUtil.executeUpdate();
daoUtil.free();

// AFTER (v8)
try (DAOUtil daoUtil = new DAOUtil(SQL_QUERY, plugin)) {
    daoUtil.setInt(1, id);
    daoUtil.executeUpdate();
}
```

## RBAC User Cast

v8 requires explicit `(User)` cast for RBAC calls:
```java
// BEFORE (v7)
RBACService.isAuthorized(resource, permission, adminUser)

// AFTER (v8)
RBACService.isAuthorized(resource, permission, (User) adminUser)
```

## Async Processing Migration

Replace `CompletableFuture.runAsync()` with Jakarta `@Asynchronous`:

```java
// BEFORE (v7)
import java.util.concurrent.CompletableFuture;
public void generateFile(IFileGenerator generator) {
    CompletableFuture.runAsync(new MyRunnable(generator));
}

// AFTER (v8)
import jakarta.enterprise.concurrent.Asynchronous;
@Asynchronous
public void generateFile(IFileGenerator generator) {
    new MyRunnable(generator).run();
}
```

## CdiHelper for Programmatic Lookup

Use `CdiHelper.getReference()` for programmatic CDI lookup with qualifiers in producers:

```java
CdiHelper.getReference(IMyService.class, "namedBeanName");
```

## Spring InitializingBean → @PostConstruct

Replace Spring's `InitializingBean.afterPropertiesSet()` with Jakarta `@PostConstruct`:

```java
// BEFORE (v7)
import org.springframework.beans.factory.InitializingBean;
public class MyComponent implements InitializingBean {
    @Override
    public void afterPropertiesSet() throws Exception {
        Assert.notNull(_field, "Required");
    }
}

// AFTER (v8)
import jakarta.annotation.PostConstruct;
public class MyComponent {
    @PostConstruct
    public void afterPropertiesSet() {
        if (_field == null) throw new IllegalArgumentException("Required");
    }
}
```

Also remove `extends InitializingBean` from interfaces.

## Workflow Task Signature Changes

If implementing `ITask.processTaskWithResult()`, update to the new signature:

```java
// BEFORE (v7)
boolean processTaskWithResult(int nIdResourceHistory, HttpServletRequest request, Locale locale, User user);

// AFTER (v8) - includes resource info
boolean processTaskWithResult(int nIdResource, String strResourceType, int nIdResourceHistory,
    HttpServletRequest request, Locale locale, User user);
```

Similarly for `AsynchronousSimpleTask.processAsynchronousTask()`.

## Verification (MANDATORY before next phase)

1. Run grep checks:
   - `grep -r "SpringContextService" src/main/java/` → must return nothing
   - `grep -r "org.springframework" src/main/java/` → must return nothing
   - `grep -r "_context.xml" webapp/` → must return nothing (files deleted)
2. Verify `beans.xml` exists at `src/main/resources/META-INF/beans.xml`
3. **No build** — phases 4-9 may still have broken references (events, cache, REST, etc.)
4. Mark task as completed ONLY when all grep checks pass
