# Phase 7: REST API Migration (if applicable)

## 7.1 Import Migration
Replace `javax.ws.rs.*` imports with `jakarta.ws.rs.*`.

## 7.2 Jersey → Jakarta JAX-RS
If the plugin uses Jersey directly:
- Replace `ResourceConfig` with standard `Application` class using `@ApplicationPath`
- Remove Jersey-specific dependencies (`jersey-server`, `jersey-spring5`, `jersey-media-*`)
- Remove manual `register()` calls — use `@Provider` auto-discovery instead
- Remove Jersey filter registrations from `plugin.xml`

```java
// BEFORE (v7) - Jersey ResourceConfig
public class MyRestConfig extends ResourceConfig {
    public MyRestConfig() {
        register(MyExceptionMapper.class);
    }
}

// AFTER (v8) - Standard JAX-RS Application
@ApplicationPath("/rest/")
public class MyRestApplication extends Application { }
```

## 7.3 REST Authentication Filter
Replace servlet-based auth filters with JAX-RS `ContainerRequestFilter`:

```java
// AFTER (v8)
@Provider
@PreMatching
@Priority(Priorities.AUTHENTICATION)
public class MyAuthFilter implements ContainerRequestFilter {
    @Inject
    private HttpServletRequest _httpRequest;

    @Inject
    @MyAuthenticatorQualifier
    private RequestAuthenticator _authenticator;

    @Override
    public void filter(ContainerRequestContext ctx) throws IOException {
        if (!_authenticator.isRequestAuthenticated(_httpRequest)) {
            ctx.abortWith(Response.status(Response.Status.UNAUTHORIZED).build());
        }
    }
}
```

## 7.4 Custom CDI Qualifier for Authenticators
```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE})
public @interface MyAuthenticatorQualifier { }
```

## 7.5 Exception Mappers
Add `@Provider` annotation for auto-discovery (no manual registration):
```java
@Provider
public class MyExceptionMapper implements ExceptionMapper<Throwable> { ... }
```

## 7.6 REST plugin.xml changes
- Remove `<filters>` section — JAX-RS auto-discovers via `@ApplicationPath`
- Remove Jersey init-params

## Verification (MANDATORY before next phase)

1. Run grep checks:
   - `grep -r "javax.ws.rs" src/main/java/` → must return nothing
   - `grep -r "org.glassfish.jersey" pom.xml` → must return nothing
2. **No build** — other phases may still have broken references
3. Mark task as completed ONLY when all grep checks pass
