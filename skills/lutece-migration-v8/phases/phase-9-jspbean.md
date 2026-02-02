# Phase 11: JspBean (Admin) Migration

JspBean classes need CDI scope and `@Named`:

```java
// BEFORE (v7)
@Controller(controllerJsp = "ManageMyPlugin.jsp", ...)
public class MyPluginJspBean extends MVCAdminJspBean { ... }

// AFTER (v8)
@SessionScoped
@Named
@Controller(controllerJsp = "ManageMyPlugin.jsp", ...)
public class MyPluginJspBean extends MVCAdminJspBean { ... }
```

## JspBean Model Injection

In v8, JspBeans can inject a `Models` helper instead of using `getModel()` or `new HashMap<>()`:

```java
import fr.paris.lutece.portal.web.cdi.mvc.Models;

@RequestScoped
@Named
@Controller(controllerJsp = "Manage.jsp", ..., securityTokenEnabled = true)
public class MyJspBean extends MVCAdminJspBean {
    @Inject
    private Models model;

    @View(value = VIEW_MANAGE, defaultView = true)
    public String getManage(HttpServletRequest request) {
        model.put(MARK_KEY, value);
        // ...
    }
}
```

## Session state removal

JspBeans are `@RequestScoped`, not session-scoped. Pass state via request parameters:
```java
// BEFORE (v7) - session field
private int _nIdTask;

// AFTER (v8) - read from request each time
int nIdTask = NumberUtils.toInt(request.getParameter(PARAMETER_TASK_ID), 0);
```

Use `redirect(request, viewName, mapParameters)` instead of `redirectView(request, viewName)` when parameters must be preserved.

## Verification (MANDATORY before next phase)

1. **No build** — full grep verification and first build happen in Phase 12
2. Verify all JspBean classes have `@SessionScoped` and `@Named` (grep check)
3. Mark task as completed ONLY when all grep checks pass
