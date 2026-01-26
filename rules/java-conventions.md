---
description: "Lutece 8 global Java constraints: jakarta imports, beans.xml, naming prefixes"
paths:
  - "**/*.java"
---

# Java Conventions — Lutece 8

## Jakarta EE Only

Java EE imports MUST use `jakarta.*`, NEVER `javax.*`:
- `jakarta.servlet.http.HttpServletRequest`
- `jakarta.enterprise.context.ApplicationScoped` / `SessionScoped` / `RequestScoped`
- `jakarta.inject.Named` / `Inject`

EXCEPTION: JDK-standard `javax.*` packages are correct and must NOT be changed: `javax.xml.transform`, `javax.xml.parsers`, `javax.crypto`, `javax.net`, `javax.sql`, `javax.naming`, `javax.cache`.

## beans.xml Required

Every plugin MUST have `src/main/webapp/WEB-INF/beans.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="https://jakarta.ee/xml/ns/jakartaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/beans_4_0.xsd"
       bean-discovery-mode="annotated">
</beans>
```

## Naming Conventions

Field prefixes: `_str` (String), `_n` (int), `_l` (long), `_b` (boolean), `_d` (double), `_ts` (Timestamp), `_date` (Date), `_list` (List).


## JavaDoc Style

Each method MUST include JavaDoc comments

## DRY KISS YAGNI
Before implementing new features, modifications, or deletions:
- Apply DRY, KISS, and YAGNI principles
- Refactor existing code if these principles are violated
- No cosmetic refactoring without reducing complexity.