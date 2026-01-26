# Phase 5: Cache Migration

If the plugin uses caching:

1. Replace EhCache 2.x API with JCache (JSR-107) or `@LuteceCache` annotation
2. Replace `net.sf.ehcache.Cache` with `javax.cache.Cache` or `Lutece107Cache`
3. Replace `new Element(key, value)` pattern with direct `cache.put(key, value)`
4. Replace `cache.get(key).getObjectValue()` with `cache.get(key)`

```java
// v8 Cache injection
@Inject
@LuteceCache(cacheName = "myCache", keyType = String.class, valueType = MyObject.class, enable = true)
private Lutece107Cache<String, MyObject> _cache;
```

## Full cache service migration pattern

```java
// BEFORE (v7)
public class MyCacheService extends AbstractCacheableService implements EventRessourceListener {
    @Override
    public void initCache() {
        super.initCache();
        ResourceEventManager.register(this);
    }
    public void addedResource(ResourceEvent event) { handleEvent(event); }
    public void deletedResource(ResourceEvent event) { handleEvent(event); }
    public void updatedResource(ResourceEvent event) { handleEvent(event); }
}

// AFTER (v8)
@ApplicationScoped
public class MyCacheService extends AbstractCacheableService<String, Object> {
    @PostConstruct
    public void initCache() {
        initCache(CACHE_NAME, String.class, Object.class);
    }
    @Override
    public void put(String key, Object value) { ... }
    @Override
    public Object get(String key) { ... }
    @Override
    public boolean remove(String key) { ... }

    // CDI observer replaces EventRessourceListener
    public void processEvent(@Observes MyEvent event) {
        if (isCacheEnable()) { resetCache(); }
    }
}
```

## Cache method name changes

| v7 | v8 |
|----|-----|
| `getFromCache(key)` | `get(key)` |
| `putInCache(key, value)` | `put(key, value)` |
| `removeKey(key)` | `remove(key)` |

## Verification (MANDATORY before next phase)

1. Run grep check: `grep -r "net.sf.ehcache" src/main/java/` → must return nothing
2. **No build** — other phases may still have broken references
3. Mark task as completed ONLY when grep check passes
