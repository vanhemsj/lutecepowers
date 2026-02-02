# Phase 2: javax → jakarta Package Renames

## Global search-and-replace for ALL Java files

| Old Import | New Import |
|-----------|-----------|
| `javax.servlet.*` | `jakarta.servlet.*` |
| `javax.validation.*` | `jakarta.validation.*` |
| `javax.annotation.PostConstruct` | `jakarta.annotation.PostConstruct` |
| `javax.annotation.PreDestroy` | `jakarta.annotation.PreDestroy` |
| `javax.inject.*` | `jakarta.inject.*` |
| `javax.enterprise.*` | `jakarta.enterprise.*` |
| `javax.ws.rs.*` | `jakarta.ws.rs.*` |

**IMPORTANT:** Only replace the imports listed in the tables above. Do NOT replace JDK-standard `javax.*` packages that have no Jakarta equivalent:

- `javax.cache.*` — JCache (JSR-107), correct as-is
- `javax.xml.transform.*` — JAXP, correct as-is
- `javax.xml.parsers.*` — SAX/DOM, correct as-is
- `javax.crypto.*` — JCE, correct as-is
- `javax.net.*` / `javax.net.ssl.*` — Networking, correct as-is
- `javax.sql.*` — JDBC, correct as-is
- `javax.naming.*` — JNDI, correct as-is

## Additional replacements

| Old Import | New Import |
|-----------|-----------|
| `javax.xml.bind.*` | `jakarta.xml.bind.*` |
| `org.apache.commons.fileupload.FileItem` | `fr.paris.lutece.portal.service.upload.MultipartItem` |

## API changes

| Old | New |
|-----|-----|
| `File.getIdFile()` / `File.setIdFile()` | `File.getFileKey()` / `File.setFileKey()` |

## MultipartItem.delete() Exception Handling

`MultipartItem.delete()` throws `IOException` (unlike `FileItem.delete()` which was unchecked):

```java
// v7
fileItem.delete();

// v8
try {
    fileItem.delete();
} catch (IOException e) {
    AppLogService.error(e.getMessage(), e);
}
```

## JSON Library Migration (if applicable)

If the plugin uses `net.sf.json-lib`, migrate to Jackson (`com.fasterxml.jackson`):

| v7 (json-lib) | v8 (Jackson) |
|---------------|-------------|
| `JSONObject json = new JSONObject()` | `ObjectMapper mapper = new ObjectMapper(); ObjectNode json = mapper.createObjectNode()` |
| `json.element("key", "value")` | `json.put("key", "value")` |
| `json.accumulate("key", obj)` | Build `ArrayNode`, add to it, then `json.set("key", arrayNode)` |
| `JSONSerializer.toJSON(obj)` | `mapper.valueToTree(obj)` |
| `jsonArray.getString(i)` | `jsonArray.get(i).asText()` |
| `json.accumulateAll(other)` | `json.setAll(otherObjectNode)` |

Remove dependency:
```xml
<!-- REMOVE -->
<dependency>
    <groupId>net.sf.json-lib</groupId>
    <artifactId>json-lib</artifactId>
</dependency>
```

## Logging Migration (in libraries)

### Log4j 1.x → Log4j 2.x (in tests/older code)
```java
// BEFORE
import org.apache.log4j.Logger;
private Logger _logger = Logger.getLogger(this.getClass());

// AFTER
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
private Logger _logger = LogManager.getLogger(this.getClass());
```

## Verification (MANDATORY before next phase)

1. Run grep checks: `grep -r "javax.servlet\|javax.validation\|javax.annotation.PostConstruct\|javax.annotation.PreDestroy\|javax.inject\|javax.ws.rs\|javax.xml.bind" src/main/java/` → must return nothing
2. Verify `javax.cache` imports are preserved (NOT replaced)
3. **No build** — Spring references still present, compilation will fail until Phase 3
4. Mark task as completed ONLY when all grep checks pass
