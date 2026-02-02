# Phase 6: Configuration Migration

If the plugin uses `AppPropertiesService` for injected config in CDI beans, optionally use MicroProfile Config:

```java
@Inject
@ConfigProperty(name = "myplugin.myProperty", defaultValue = "default")
private String _myProperty;
```

**Note:** `AppPropertiesService.getProperty()` still works in v8. MicroProfile Config is optional but preferred in CDI beans.

## AppPropertiesService → MicroProfile Config (in libraries)

For libraries that don't have CDI injection context, use `ConfigProvider.getConfig()` directly:

```java
// BEFORE (v7)
import fr.paris.lutece.portal.service.util.AppPropertiesService;
String value = AppPropertiesService.getProperty(PROPERTY_KEY);
String valueWithDefault = AppPropertiesService.getProperty(PROPERTY_KEY, "default");

// AFTER (v8)
import org.eclipse.microprofile.config.Config;
import org.eclipse.microprofile.config.ConfigProvider;
private static Config _config = ConfigProvider.getConfig();
String value = _config.getOptionalValue(PROPERTY_KEY, String.class).orElse(null);
String valueWithDefault = _config.getOptionalValue(PROPERTY_KEY, String.class).orElse("default");
```

## Verification (MANDATORY before next phase)

1. Verify `@ConfigProperty` replacements are correct (if any were made)
2. **No build** — other phases may still have broken references
3. Mark task as completed
