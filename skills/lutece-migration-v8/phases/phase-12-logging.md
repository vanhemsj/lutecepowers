# Phase 12: Logging Migration

Update string concatenation to parameterized logging:

```java
// BEFORE (v7)
AppLogService.info(MyClass.class.getName() + " : message " + variable);

// AFTER (v8)
AppLogService.info("{} : message {}", MyClass.class.getName(), variable);
```

## Verification (MANDATORY before next phase)

1. **No build** — first build happens in Phase 13
2. Run grep check: `grep -rn "AppLogService\.\(info\|error\|debug\|warn\).*+ " src/main/java/` → should return nothing (no string concatenation in log calls)
3. Mark task as completed ONLY when grep check passes
