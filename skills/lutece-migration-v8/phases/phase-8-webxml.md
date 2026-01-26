# Phase 8: Web.xml & Plugin Descriptor Updates

## 8.1 Web.xml (if applicable)

Mainly for plugins with web.xml fragments.

1. Update namespace from `http://java.sun.com/xml/ns/javaee` to `https://jakarta.ee/xml/ns/jakartaee`
2. Update version to `6.0`
3. Remove Spring listeners if present

## 8.2 Plugin Descriptor (`plugin.xml` or `pluginName.xml`)

1. Update `<version>` to the v8 version
2. Update `<min-core-version>` to `8.0.0`
3. **Remove `<application-class>`** from `<application>` elements — XPages are auto-discovered via CDI
4. Update `<icon-url>` if new icon paths are used

```xml
<!-- BEFORE -->
<application>
    <application-id>myPlugin</application-id>
    <application-class>fr.paris.lutece.plugins.myplugin.web.MyXPage</application-class>
</application>

<!-- AFTER -->
<application>
    <application-id>myPlugin</application-id>
    <!-- No application-class: auto-discovered via CDI -->
</application>
```

## Verification (MANDATORY before next phase)

1. Run grep checks:
   - `grep -r "java.sun.com/xml/ns/javaee" webapp/` → must return nothing
   - `grep -rn "<application-class>" webapp/WEB-INF/plugins/` → must return nothing
2. Verify `<min-core-version>` is `8.0.0` in plugin descriptor
3. **No build** — other phases may still have broken references
4. Mark task as completed ONLY when grep checks pass
