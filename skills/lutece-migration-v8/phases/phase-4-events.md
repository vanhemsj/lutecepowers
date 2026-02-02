# Phase 4: Event/Listener Migration

## 4.1 CDI Observer Pattern

Replace custom listener interfaces with CDI `@Observes`:

```java
// BEFORE: listener interface + registration
public interface MyEventListener {
    void processEvent(MyEvent event);
}

// AFTER: CDI observer
@ApplicationScoped
public class MyEventObserver {
    public void processEvent(@Observes MyEvent event) {
        // Handle event
    }
}
```

## 4.2 Firing Events

```java
// BEFORE
for (MyListener l : SpringContextService.getBeansOfType(MyListener.class)) {
    l.onEvent(event);
}

// AFTER
CDI.current().getBeanManager().getEvent().fire(event);
```

## CDI Events with TypeQualifier

For fine-grained event filtering, use `@Type(EventAction.*)` qualifier:

```java
// Firing with qualifier
CDI.current().getBeanManager().getEvent()
    .select(MyEvent.class, new TypeQualifier(EventAction.CREATE))
    .fire(event);

// Observing with qualifier (async)
public void onCreated(@ObservesAsync @Type(EventAction.CREATE) MyEvent event) { ... }
```

| Action | EventAction |
|--------|------------|
| Create | `EventAction.CREATE` |
| Update | `EventAction.UPDATE` |
| Delete | `EventAction.REMOVE` |

## Verification (MANDATORY before next phase)

1. Run grep check: `grep -r "SpringContextService.getBeansOfType" src/main/java/` → must return nothing
2. **No build** — other phases may still have broken references
3. Mark task as completed ONLY when grep check passes
