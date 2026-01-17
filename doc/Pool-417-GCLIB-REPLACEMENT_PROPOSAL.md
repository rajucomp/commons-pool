
# Proposal: Replace CGLib with ByteBuddy in Apache Commons Pool

**Author:** Raju Gupta
**Date:** January 2026  
**Status:** Draft Proposal  
**Affects:** `commons-pool3` proxy package

---

## Executive Summary

CGLib is unmaintained and incompatible with JDK 17+. This proposal recommends adding ByteBuddy as a modern alternative for class-based proxying while deprecating CGLib classes for removal in version 4.0.0.

**Recommendation:** Proceed with ByteBuddy integration in v3.0.0 with a gradual deprecation strategy.

**Key Facts:**
- CGLib's own maintainers recommend migrating to ByteBuddy
- Current CGLib usage is minimal: 2 classes, ~190 lines of code
- Estimated implementation effort: **6-8 developer days**
- No breaking changes until v4.0.0

---

## Problem Statement

The CGLib dependency presents several issues:

1. **Unmaintained**: Last release (3.3.0) was August 2019
2. **JDK Compatibility**: Requires `--add-opens` flags on JDK 17; broken on JDK 21+
3. **Security Risk**: No patches for potential vulnerabilities
4. **User Impact**: Projects using `CglibProxySource` cannot upgrade to modern JDKs

From CGLib's README:
> *"cglib is unmaintained and does not work well (or possibly at all?) in newer JDKs, particularly JDK17+... you'll probably have better luck migrating to something like ByteBuddy."*

---

## Proposed Solution

### Approach: Gradual Deprecation with ByteBuddy Alternative

| Phase | Version | Actions |
|-------|---------|---------|
| **1** | v3.0.0 | Add `@Deprecated` to CGLib classes; add ByteBuddy implementation |
| **2** | v3.1.0 | Add runtime warning when CGLib classes are instantiated |
| **3** | v4.0.0 | Remove CGLib classes and dependency |

### Why ByteBuddy?

- **Recommended by CGLib maintainers** as the migration target
- **Actively maintained** with regular releases
- **Full JDK 17/21/25+ support** without workarounds
- **Similar API pattern** - minimal user migration effort
- **Proven adoption** - used by Mockito, Hibernate, Spring

---

## Implementation Details

### New Classes

Two new classes following the existing architecture:

**ByteBuddyProxySource.java** (extends `AbstractProxySource<T>`)

```java
public class ByteBuddyProxySource<T> extends AbstractProxySource<T> {
    
    private final Class<? extends T> superclass;
    
    public ByteBuddyProxySource(Class<? extends T> superclass) {
        super(builder());
        this.superclass = superclass;
    }
    
    @Override
    public T createProxy(T pooledObject, UsageTracking<T> usageTracking) {
        // ByteBuddy subclass creation with MethodDelegation
    }
    
    @Override
    public T resolveProxy(T proxy) {
        // Extract handler and disable proxy
    }
}
```

**ByteBuddyProxyHandler.java** (extends `BaseProxyHandler<T>`)

```java
final class ByteBuddyProxyHandler<T> extends BaseProxyHandler<T> {
    
    @RuntimeType
    public Object intercept(@This Object self,
                           @Origin Method method,
                           @AllArguments Object[] args) throws Throwable {
        return doInvoke(method, args);
    }
}
```

### Deprecation Annotations

```java
/**
 * @deprecated CGLib is unmaintained and incompatible with JDK 21+.
 *             Use {@link ByteBuddyProxySource} or {@link JdkProxySource} instead.
 *             Will be removed in version 4.0.0.
 */
@Deprecated(since = "3.0.0", forRemoval = true)
public class CglibProxySource<T> extends AbstractProxySource<T> {
```

### Build Changes (pom.xml)

New optional dependency:

```xml
<dependency>
    <groupId>net.bytebuddy</groupId>
    <artifactId>byte-buddy</artifactId>
    <version>1.14.18</version>
    <optional>true</optional>
</dependency>
```

Updated OSGi imports:

```xml
<commons.osgi.import>
    net.sf.cglib.proxy;resolution:=optional,
    net.bytebuddy.*;resolution:=optional,
    *
</commons.osgi.import>
```

---

## User Migration Path

### For Interface-Based Proxying (No Change Required)

Users proxying interfaces should already use `JdkProxySource`:

```java
// Recommended - zero external dependencies
ProxySource<MyInterface> source = new JdkProxySource<>(
    classLoader, new Class<?>[] { MyInterface.class });
```

### For Class-Based Proxying

**Before (CGLib):**

```java
ProxySource<MyClass> source = new CglibProxySource<>(MyClass.class);
```

**After (ByteBuddy):**

```java
ProxySource<MyClass> source = new ByteBuddyProxySource<>(MyClass.class);
```

The API is intentionally similar to minimize migration effort.

---

## Effort Estimation

| Task | Effort | Notes |
|------|--------|-------|
| Implement `ByteBuddyProxySource` | 2-3 days | Including Builder pattern |
| Implement `ByteBuddyProxyHandler` | 1 day | Leverages `BaseProxyHandler` |
| Unit tests | 1-2 days | Mirror existing CGLib tests |
| Documentation updates | 1 day | Javadoc, package-info |
| Build configuration | 0.5 days | pom.xml, OSGi |
| **Total** | **6-8 days** | |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| ByteBuddy API changes | Low | Medium | Pin to stable version; ByteBuddy has excellent backwards compatibility |
| User migration friction | Medium | Low | Similar API; clear documentation; 2+ version deprecation period |
| CGLib breaks before v4.0.0 | Medium | Medium | ByteBuddy alternative immediately available |

---

## JDK Compatibility Summary

| JDK | CGLib | ByteBuddy | JDK Proxy |
|-----|-------|-----------|-----------|
| 8 | ✅ | ✅ | ✅ |
| 11 | ✅ | ✅ | ✅ |
| 17 | ⚠️ `--add-opens` | ✅ | ✅ |
| 21 | ❌ | ✅ | ✅ |
| 25+ | ❌ | ✅ | ✅ |

---

## Backwards Compatibility

- **Binary Compatible**: Yes - new classes are additions
- **Source Compatible**: Yes - existing code continues to compile with deprecation warnings
- **Behavioral Compatible**: Yes - ByteBuddy proxies behave identically to CGLib proxies

**Breaking change occurs only in v4.0.0** when CGLib classes are removed.

---

## Decision Points for Maintainers

1. **Proceed with ByteBuddy?**
    - Alternative: Support only `JdkProxySource` (loses class proxying capability)
    - Recommendation: Yes - ByteBuddy maintains feature parity

2. **Deprecation Timeline?**
    - Proposed: Deprecate in v3.0.0, remove in v4.0.0
    - Alternative: Faster removal in v3.1.0 (less migration time for users)

3. **Runtime Warning?**
    - Proposed: Add warning log in v3.1.0 when CGLib classes are used
    - Alternative: Skip warning, rely on compile-time deprecation only

---

## Open Questions

1. Should we add a `ProxySourceFactory` to auto-select the best available implementation?
2. Should ByteBuddy version be configurable via a property?
3. Do we need integration tests against JDK 21+ in CI?

---

## References

- [CGLib GitHub - Unmaintained Notice](https://github.com/cglib/cglib)
- [ByteBuddy Documentation](https://bytebuddy.net)
- [ByteBuddy Proxy Creation Guide](http://mydailyjava.blogspot.com/2022/02/using-byte-buddy-for-proxy-creation.html)
- Current implementation: `src/main/java/org/apache/commons/pool3/proxy/`

---

## Requested Action

Please review and provide feedback on:
1. Agreement to proceed with ByteBuddy integration
2. Approval of the proposed deprecation timeline
3. Any concerns about backwards compatibility approach

I am prepared to implement this proposal upon approval.