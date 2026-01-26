# Phase 10: JSP Migration

All JSP files must be migrated from scriptlet-based (`<jsp:useBean>` + `<% %>`) to EL-based (`${ }`) syntax using CDI bean names.

## Bean name resolution

The CDI bean name is the **camelCase** of the class name (or the `@Named` value if specified):
- `AdminWorkgroupJspBean` → `adminWorkgroupJspBean`
- `TemporaryFilesJspBean` → `temporaryFilesJspBean`

## Pattern 1: View JSP (displays a page)

Used for Manage, Create, Modify pages that render HTML content.

```jsp
<%-- BEFORE (v7) --%>
<%@ page errorPage="../../ErrorPage.jsp" %>
<jsp:useBean id="myPluginJspBean" scope="session"
    class="fr.paris.lutece.plugins.myplugin.web.MyPluginJspBean" />
<jsp:include page="../../AdminHeader.jsp" />
<% myPluginJspBean.init( request, right ); %>
<%= myPluginJspBean.getManageItems( request ) %>
<%@ include file="../../AdminFooter.jsp" %>

<%-- AFTER (v8) --%>
<%@ page errorPage="../../ErrorPage.jsp" %>
<jsp:include page="../../AdminHeader.jsp" />

<%@page import="fr.paris.lutece.plugins.myplugin.web.MyPluginJspBean"%>

${ myPluginJspBean.init( pageContext.request, MyPluginJspBean.RIGHT_MANAGE ) }
${ myPluginJspBean.getManageItems( pageContext.request ) }

<%@ include file="../../AdminFooter.jsp" %>
```

## Pattern 2: Action JSP (Do — executes an action and redirects)

Used for DoCreate, DoModify, DoRemove pages that process form data and redirect.

```jsp
<%-- BEFORE (v7) --%>
<%@ page errorPage="../../ErrorPage.jsp" %>
<jsp:useBean id="myPluginJspBean" scope="session"
    class="fr.paris.lutece.plugins.myplugin.web.MyPluginJspBean" />
<% response.sendRedirect( myPluginJspBean.doCreateItem( request ) ); %>

<%-- AFTER (v8) --%>
<%@ page errorPage="../../ErrorPage.jsp" %>

<%@page import="fr.paris.lutece.plugins.myplugin.web.MyPluginJspBean"%>

${ myPluginJspBean.init( pageContext.request, MyPluginJspBean.RIGHT_MANAGE ) }
${ pageContext.response.sendRedirect( myPluginJspBean.doCreateItem( pageContext.request ) ) }
```

## Pattern 3: ProcessController JSP (legacy controller dispatch)

Used when the JSP delegates to `processController()` which returns HTML content.

```jsp
<%-- BEFORE (v7) --%>
<%@ page errorPage="../../ErrorPage.jsp" %>
<jsp:useBean id="myPluginJspBean" scope="session"
    class="fr.paris.lutece.plugins.myplugin.web.MyPluginJspBean" />
<jsp:include page="../../AdminHeader.jsp" />
<% String strContent = myPluginJspBean.processController( request, response ); %>
<%= strContent %>
<%@ include file="../../AdminFooter.jsp" %>

<%-- AFTER (v8) --%>
<%@ page errorPage="../../ErrorPage.jsp" %>
<jsp:include page="../../AdminHeader.jsp" />

${ pageContext.setAttribute( 'strContent', myPluginJspBean.processController( pageContext.request, pageContext.response ) ) }
${ pageContext.getAttribute( 'strContent' ) }

<%@ include file="../../AdminFooter.jsp" %>
```

## Pattern 4: Action JSP with response handling (download, etc.)

Used for actions that write directly to the response (file download, etc.).

```jsp
<%-- AFTER (v8) --%>
<%@ page errorPage="../../ErrorPage.jsp" %>

<%@page import="fr.paris.lutece.plugins.myplugin.web.MyPluginJspBean"%>

${ myPluginJspBean.init( pageContext.request, MyPluginJspBean.VIEW_FILES ) }
${ myPluginJspBean.doDownloadFile( pageContext.request, pageContext.response ) }
```

## JSP migration rules

1. **Remove** all `<jsp:useBean>` tags — beans are CDI-managed, not session-scoped JSP beans
2. **Add** `<%@page import="...JspBean"%>` to reference the class constants (RIGHT_xxx, VIEW_xxx)
3. **Replace** `request` → `pageContext.request`, `response` → `pageContext.response`
4. **Replace** scriptlets `<% %>` and expressions `<%= %>` with EL `${ }`
5. **Add** `init()` call with the right constant before any bean method call
6. **Keep** `errorPage`, `AdminHeader.jsp` include, and `AdminFooter.jsp` include unchanged

## Verification (MANDATORY before next phase)

1. **No build** — JspBeans are not yet migrated, compilation will fail until Phase 12
2. Run grep checks:
   - `grep -r "jsp:useBean" webapp/` → must return nothing
   - `grep -r "<%" webapp/jsp/` → check no old scriptlets remain (except `<%@` directives)
3. Mark task as completed ONLY when all grep checks pass
