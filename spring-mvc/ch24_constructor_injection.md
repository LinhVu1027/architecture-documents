# Chapter 24: Constructor Injection / @Autowired / @Value

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The BeanContainer stores pre-instantiated singletons. ClasspathScanner instantiates beans via no-arg constructors and registers them immediately. | Components cannot depend on each other — a `UserController` can't receive a `UserService` through its constructor. Every bean must be self-contained with a no-arg constructor. | Upgrade the container to resolve and inject dependencies automatically — constructor parameters, `@Autowired` fields/methods, and externalized `@Value` properties — transforming the registry into a true dependency injection engine. |

---

## 24.1 The Integration Point

The integration point is **`SimpleBeanContainer`** (ch01) — specifically, the gap between *registration* and *retrieval*. Currently, `registerBean(name, instance)` stores a fully-constructed object. There is no lifecycle between "I know this class exists" and "here's a live instance."

**Before (ch01–ch23):**

```java
// ClasspathScanner: discovers class, instantiates immediately, registers
Object instance = clazz.getDeclaredConstructor().newInstance(); // must be no-arg!
beanContainer.registerBean(beanName, instance);

// SimpleBeanContainer: just a map
singletonMap.put(name, bean);
```

**After (ch24):**

```java
// ClasspathScanner: discovers class, registers definition only
beanContainer.registerBeanDefinition(beanName, clazz);

// SimpleBeanContainer.refresh(): instantiates with full DI lifecycle
ConstructorResolver resolver = new ConstructorResolver(this);
Object instance = resolver.autowireConstructor(definition.getBeanClass());  // resolves constructor params
singletonMap.put(beanName, instance);

for (BeanPostProcessor bpp : beanPostProcessors) {
    instance = bpp.postProcessBeforeInitialization(instance, beanName);     // @Autowired/@Value injection
}
```

Two key decisions here:

1. **Separate registration from instantiation.** `registerBeanDefinition(name, class)` records *what* to create. `refresh()` creates everything in dependency order. This separation is what enables constructor injection — you can't inject a dependency into a constructor if you've already called that constructor. This matches the real framework's `BeanDefinition` vs `singletonObjects` pattern.

2. **`BeanPostProcessor` as the extension point** for field/method injection. Constructor injection is handled directly during instantiation, but field/method injection happens in a separate phase via `AutowiredAnnotationBeanPostProcessor`. This split matches the real framework's lifecycle and keeps injection logic pluggable.

**Direction:** This connects **BeanDefinition metadata** (class info, not yet instantiated) to **live singleton instances** (fully wired). To make it work, we need:
- `@Autowired`, `@Value`, `@Qualifier` annotations — markers for injection points
- `BeanPostProcessor` interface — lifecycle hook after construction
- `BeanDefinition` — class metadata stored before instantiation
- `PropertySource` — externalized config for `@Value` resolution
- `ConstructorResolver` — selects constructor, resolves parameters, instantiates
- `InjectionMetadata` — holds discovered `@Autowired`/`@Value` fields and methods
- `AutowiredAnnotationBeanPostProcessor` — scans and injects after construction
- Updated `SimpleBeanContainer` — two-phase lifecycle with DI
- Updated `ClasspathScanner` — registers definitions instead of instances

## 24.2 The Annotations

Three annotations mark injection points.

**New file:** `src/main/java/com/simplespringmvc/injection/Autowired.java`

```java
package com.simplespringmvc.injection;

import java.lang.annotation.*;

@Target({ElementType.CONSTRUCTOR, ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {
}
```

**New file:** `src/main/java/com/simplespringmvc/injection/Value.java`

```java
package com.simplespringmvc.injection;

import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {
    String value();
}
```

**New file:** `src/main/java/com/simplespringmvc/injection/Qualifier.java`

```java
package com.simplespringmvc.injection;

import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Qualifier {
    String value();
}
```

## 24.3 The `BeanPostProcessor` Interface

The central extension point for intercepting bean creation. Both methods have `default` implementations returning the bean unchanged — implementations override only what they need.

**New file:** `src/main/java/com/simplespringmvc/injection/BeanPostProcessor.java`

```java
package com.simplespringmvc.injection;

public interface BeanPostProcessor {

    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

In the real framework, `postProcessBeforeInitialization` is where `@Autowired` field/method injection happens. `postProcessAfterInitialization` is where AOP proxies get created (future ch28). Our `AutowiredAnnotationBeanPostProcessor` implements the "before" hook.

## 24.4 Supporting Types

**New file:** `src/main/java/com/simplespringmvc/injection/BeanDefinition.java`

```java
package com.simplespringmvc.injection;

public class BeanDefinition {
    private final String beanName;
    private final Class<?> beanClass;

    public BeanDefinition(String beanName, Class<?> beanClass) {
        this.beanName = beanName;
        this.beanClass = beanClass;
    }

    public String getBeanName() { return beanName; }
    public Class<?> getBeanClass() { return beanClass; }
}
```

The real `BeanDefinition` stores scope, lazy-init flag, constructor args, factory method, init/destroy callbacks, and more. Ours holds just the name and class — enough to demonstrate that **separating registration from instantiation** is what unlocks dependency ordering.

**New file:** `src/main/java/com/simplespringmvc/injection/PropertySource.java`

```java
package com.simplespringmvc.injection;

import java.util.Collections;
import java.util.Map;

public class PropertySource {
    private final Map<String, String> properties;

    public PropertySource(Map<String, String> properties) {
        this.properties = Map.copyOf(properties);
    }

    public String getProperty(String key) {
        return properties.get(key);
    }

    /**
     * Resolve a @Value expression:
     *   "${key}"         → required lookup
     *   "${key:default}" → optional with default
     *   "literal"        → returned as-is
     */
    public String resolveValue(String expression) {
        if (!expression.startsWith("${") || !expression.endsWith("}")) {
            return expression;  // literal
        }

        String placeholder = expression.substring(2, expression.length() - 1);
        int separatorIndex = placeholder.indexOf(':');

        if (separatorIndex >= 0) {
            String key = placeholder.substring(0, separatorIndex);
            String defaultValue = placeholder.substring(separatorIndex + 1);
            String value = properties.get(key);
            return value != null ? value : defaultValue;
        }

        String value = properties.get(placeholder);
        if (value == null) {
            throw new IllegalArgumentException(
                    "Could not resolve placeholder '${" + placeholder + "}'");
        }
        return value;
    }

    public Map<String, String> getAll() {
        return Collections.unmodifiableMap(properties);
    }
}
```

**New file:** `src/main/java/com/simplespringmvc/injection/CircularDependencyException.java`

```java
package com.simplespringmvc.injection;

public class CircularDependencyException extends RuntimeException {
    public CircularDependencyException(String message) {
        super(message);
    }
}
```

**New file:** `src/main/java/com/simplespringmvc/injection/UnsatisfiedDependencyException.java`

```java
package com.simplespringmvc.injection;

public class UnsatisfiedDependencyException extends RuntimeException {
    public UnsatisfiedDependencyException(String message) { super(message); }
    public UnsatisfiedDependencyException(String message, Throwable cause) { super(message, cause); }
}
```

## 24.5 The `ConstructorResolver`

Selects the right constructor and resolves each parameter from the container.

**New file:** `src/main/java/com/simplespringmvc/injection/ConstructorResolver.java`

```java
package com.simplespringmvc.injection;

import com.simplespringmvc.container.SimpleBeanContainer;
import java.lang.reflect.Constructor;
import java.lang.reflect.Parameter;

public class ConstructorResolver {
    private final SimpleBeanContainer container;

    public ConstructorResolver(SimpleBeanContainer container) {
        this.container = container;
    }

    public Object autowireConstructor(Class<?> beanClass) {
        Constructor<?> constructor = determineConstructor(beanClass);
        Object[] args = resolveArguments(constructor);
        try {
            constructor.setAccessible(true);
            return constructor.newInstance(args);
        } catch (ReflectiveOperationException e) {
            throw new UnsatisfiedDependencyException(
                    "Failed to instantiate " + beanClass.getName(), e);
        }
    }
```

**Constructor selection algorithm:**

```java
    Constructor<?> determineConstructor(Class<?> beanClass) {
        Constructor<?>[] constructors = beanClass.getDeclaredConstructors();

        // 1. Find @Autowired constructor
        Constructor<?> autowiredConstructor = null;
        int autowiredCount = 0;
        for (Constructor<?> ctor : constructors) {
            if (ctor.isAnnotationPresent(Autowired.class)) {
                autowiredConstructor = ctor;
                autowiredCount++;
            }
        }

        if (autowiredCount > 1) {
            throw new UnsatisfiedDependencyException(
                    "Multiple @Autowired constructors found on " + beanClass.getName());
        }
        if (autowiredConstructor != null) {
            return autowiredConstructor;
        }

        // 2. Single constructor auto-detection
        if (constructors.length == 1) {
            return constructors[0];
        }

        // 3. Multiple constructors — fall back to no-arg
        for (Constructor<?> ctor : constructors) {
            if (ctor.getParameterCount() == 0) {
                return ctor;
            }
        }

        throw new UnsatisfiedDependencyException(
                "No suitable constructor found for " + beanClass.getName());
    }
```

**Parameter resolution** — each parameter can be a bean dependency, a `@Value`, or a `@Qualifier`-disambiguated bean:

```java
    Object[] resolveArguments(Constructor<?> constructor) {
        Parameter[] parameters = constructor.getParameters();
        Object[] args = new Object[parameters.length];
        for (int i = 0; i < parameters.length; i++) {
            args[i] = resolveParameter(parameters[i], constructor);
        }
        return args;
    }

    private Object resolveParameter(Parameter parameter, Constructor<?> constructor) {
        // @Value → resolve from PropertySource
        Value valueAnn = parameter.getAnnotation(Value.class);
        if (valueAnn != null) {
            return resolveValueParameter(valueAnn.value(), parameter.getType());
        }

        // @Qualifier → resolve bean by name
        String qualifier = null;
        Qualifier qualifierAnn = parameter.getAnnotation(Qualifier.class);
        if (qualifierAnn != null) {
            qualifier = qualifierAnn.value();
        }

        try {
            return container.resolveDependency(parameter.getType(), qualifier);
        } catch (CircularDependencyException e) {
            throw e;
        } catch (Exception e) {
            throw new UnsatisfiedDependencyException(
                    "Failed to resolve parameter '" + parameter.getName() + "' of type "
                            + parameter.getType().getName() + " in constructor of "
                            + constructor.getDeclaringClass().getName(), e);
        }
    }
}
```

## 24.6 The `InjectionMetadata` and Inner Element Classes

Holds the injection points discovered on a class — `@Autowired` fields, `@Autowired` methods, and `@Value` fields.

**New file:** `src/main/java/com/simplespringmvc/injection/InjectionMetadata.java`

```java
package com.simplespringmvc.injection;

import com.simplespringmvc.container.SimpleBeanContainer;
import java.lang.reflect.*;
import java.util.List;

public class InjectionMetadata {
    private static final InjectionMetadata EMPTY = new InjectionMetadata(Object.class, List.of());

    private final Class<?> targetClass;
    private final List<InjectedElement> elements;

    public InjectionMetadata(Class<?> targetClass, List<InjectedElement> elements) {
        this.targetClass = targetClass;
        this.elements = List.copyOf(elements);
    }

    public static InjectionMetadata forElements(List<InjectedElement> elements, Class<?> clazz) {
        return elements.isEmpty() ? EMPTY : new InjectionMetadata(clazz, elements);
    }

    public void inject(Object target, String beanName, SimpleBeanContainer container) {
        for (InjectedElement element : elements) {
            element.inject(target, container);
        }
    }
```

Three inner classes handle the three injection types:

```java
    // @Autowired field → resolve bean by type, set via reflection
    public static class AutowiredFieldElement extends InjectedElement {
        private final Field field;
        private final String qualifier;

        public void inject(Object target, SimpleBeanContainer container) {
            Object value = container.resolveDependency(field.getType(), qualifier);
            field.setAccessible(true);
            field.set(target, value);
        }
    }

    // @Autowired method → resolve each parameter by type, invoke
    public static class AutowiredMethodElement extends InjectedElement {
        private final Method method;

        public void inject(Object target, SimpleBeanContainer container) {
            Parameter[] params = method.getParameters();
            Object[] args = new Object[params.length];
            for (int i = 0; i < params.length; i++) {
                String q = params[i].isAnnotationPresent(Qualifier.class)
                        ? params[i].getAnnotation(Qualifier.class).value() : null;
                args[i] = container.resolveDependency(params[i].getType(), q);
            }
            method.setAccessible(true);
            method.invoke(target, args);
        }
    }

    // @Value field → resolve from PropertySource, convert type
    public static class ValueFieldElement extends InjectedElement {
        private final Field field;
        private final String expression;

        public void inject(Object target, SimpleBeanContainer container) {
            PropertySource props = container.getPropertySource();
            String resolved = props.resolveValue(expression);
            Object converted = convertIfNecessary(resolved, field.getType(), container);
            field.setAccessible(true);
            field.set(target, converted);
        }
    }
}
```

## 24.7 The `AutowiredAnnotationBeanPostProcessor`

The core BPP that discovers `@Autowired` and `@Value` injection points on a class and performs the injection.

**New file:** `src/main/java/com/simplespringmvc/injection/AutowiredAnnotationBeanPostProcessor.java`

```java
package com.simplespringmvc.injection;

import com.simplespringmvc.container.SimpleBeanContainer;
import java.lang.reflect.*;
import java.util.ArrayList;
import java.util.List;

public class AutowiredAnnotationBeanPostProcessor implements BeanPostProcessor {
    private final SimpleBeanContainer container;

    public AutowiredAnnotationBeanPostProcessor(SimpleBeanContainer container) {
        this.container = container;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        InjectionMetadata metadata = buildInjectionMetadata(bean.getClass());
        metadata.inject(bean, beanName, container);
        return bean;
    }

    InjectionMetadata buildInjectionMetadata(Class<?> clazz) {
        List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
        Class<?> targetClass = clazz;

        do {
            List<InjectionMetadata.InjectedElement> currentElements = new ArrayList<>();

            for (Field field : targetClass.getDeclaredFields()) {
                if (Modifier.isStatic(field.getModifiers())) continue;

                if (field.isAnnotationPresent(Autowired.class)) {
                    String qualifier = field.isAnnotationPresent(Qualifier.class)
                            ? field.getAnnotation(Qualifier.class).value() : null;
                    currentElements.add(
                            new InjectionMetadata.AutowiredFieldElement(field, qualifier));
                } else if (field.isAnnotationPresent(Value.class)) {
                    currentElements.add(
                            new InjectionMetadata.ValueFieldElement(
                                    field, field.getAnnotation(Value.class).value()));
                }
            }

            for (Method method : targetClass.getDeclaredMethods()) {
                if (Modifier.isStatic(method.getModifiers())) continue;
                if (method.isBridge()) continue;

                if (method.isAnnotationPresent(Autowired.class)) {
                    currentElements.add(new InjectionMetadata.AutowiredMethodElement(method));
                }
            }

            // Parent-first ordering — prepend superclass elements
            elements.addAll(0, currentElements);
            targetClass = targetClass.getSuperclass();
        } while (targetClass != null && targetClass != Object.class);

        return InjectionMetadata.forElements(elements, clazz);
    }
}
```

The class hierarchy walk (target → superclass → ... → Object) matches the real `AutowiredAnnotationBeanPostProcessor.buildAutowiringMetadata()` at line 547. Parent elements are prepended so that a parent's `@Autowired` fields are injected before the child's — the same order as the real framework.

## 24.8 Upgrading `SimpleBeanContainer`

The largest change: the container gains a two-phase lifecycle.

**Modifying:** `src/main/java/com/simplespringmvc/container/SimpleBeanContainer.java`

**New fields:**

```java
// Bean definitions awaiting instantiation
private final Map<String, BeanDefinition> beanDefinitions = new LinkedHashMap<>();

// Lifecycle hooks applied to every bean created during refresh
private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<>();

// Circular dependency detection — bean names currently being created
private final Set<String> currentlyCreating = new HashSet<>();

// Externalized config for @Value
private PropertySource propertySource = new PropertySource(Map.of());
```

**New method: `registerBeanDefinition()`**

```java
public void registerBeanDefinition(String name, Class<?> beanClass) {
    beanDefinitions.put(name, new BeanDefinition(name, beanClass));
    if (!beanNames.contains(name)) {
        beanNames.add(name);
    }
}
```

**New method: `refresh()` — the lifecycle orchestrator**

```java
public void refresh() {
    // 1. Auto-register the @Autowired/@Value processor
    boolean hasAutowiredProcessor = beanPostProcessors.stream()
            .anyMatch(bpp -> bpp instanceof AutowiredAnnotationBeanPostProcessor);
    if (!hasAutowiredProcessor) {
        beanPostProcessors.add(new AutowiredAnnotationBeanPostProcessor(this));
    }

    // 2. Instantiate BeanPostProcessor definitions first (infrastructure)
    for (var entry : new ArrayList<>(beanDefinitions.entrySet())) {
        if (BeanPostProcessor.class.isAssignableFrom(entry.getValue().getBeanClass())) {
            Object bpp = createBean(entry.getValue());
            if (bpp instanceof BeanPostProcessor processor) {
                addBeanPostProcessor(processor);
            }
        }
    }

    // 3. Instantiate all remaining bean definitions
    for (var entry : new ArrayList<>(beanDefinitions.entrySet())) {
        if (!singletonMap.containsKey(entry.getKey())) {
            createBean(entry.getValue());
        }
    }
}
```

**New method: `createBean()` — full DI lifecycle for a single bean**

```java
private Object createBean(BeanDefinition definition) {
    String beanName = definition.getBeanName();

    // Already created?
    Object existing = singletonMap.get(beanName);
    if (existing != null) return existing;

    // Circular dependency detection
    if (currentlyCreating.contains(beanName)) {
        throw new CircularDependencyException(
                "Circular dependency detected: bean '" + beanName + "' is currently being created.");
    }

    currentlyCreating.add(beanName);
    try {
        // Phase 1: Constructor resolution + instantiation
        ConstructorResolver resolver = new ConstructorResolver(this);
        Object instance = resolver.autowireConstructor(definition.getBeanClass());
        singletonMap.put(beanName, instance);  // register early for field injection deps

        // Phase 2: @Autowired/@Value field/method injection
        for (BeanPostProcessor bpp : beanPostProcessors) {
            instance = bpp.postProcessBeforeInitialization(instance, beanName);
        }

        // Phase 3: Post-initialization (future AOP proxies)
        for (BeanPostProcessor bpp : beanPostProcessors) {
            instance = bpp.postProcessAfterInitialization(instance, beanName);
        }

        singletonMap.put(beanName, instance);
        return instance;
    } finally {
        currentlyCreating.remove(beanName);
    }
}
```

**New method: `resolveDependency()` — the central dependency resolution**

```java
public Object resolveDependency(Class<?> type, String qualifier) {
    Map<String, Object> candidates = new LinkedHashMap<>();

    // 1. Check existing singletons
    for (String name : beanNames) {
        Object bean = singletonMap.get(name);
        if (bean != null && type.isInstance(bean)) {
            candidates.put(name, bean);
        }
    }

    // 2. Check uninstantiated definitions → create on-demand
    for (var entry : beanDefinitions.entrySet()) {
        String name = entry.getKey();
        if (!singletonMap.containsKey(name)
                && type.isAssignableFrom(entry.getValue().getBeanClass())) {
            Object bean = createBean(entry.getValue());
            candidates.put(name, bean);
        }
    }

    // 3. Apply @Qualifier filter
    if (qualifier != null && !qualifier.isEmpty()) {
        Object qualified = candidates.get(qualifier);
        if (qualified != null) return qualified;
        throw new UnsatisfiedDependencyException("No bean named '" + qualifier + "'");
    }

    // 4. Exactly one match
    if (candidates.size() == 1) return candidates.values().iterator().next();
    if (candidates.isEmpty()) throw new UnsatisfiedDependencyException("No bean of type " + type.getName());
    throw new NoUniqueBeanException("Multiple beans of type " + type.getName() + ": " + candidates.keySet());
}
```

**Modified method: `containsBean()`** now also checks bean definitions:

```java
@Override
public boolean containsBean(String name) {
    return singletonMap.containsKey(name) || beanDefinitions.containsKey(name);
}
```

**Modified method: `getBean(String)`** now triggers on-demand creation from definitions:

```java
@Override
public Object getBean(String name) {
    Object bean = singletonMap.get(name);
    if (bean != null) return bean;

    BeanDefinition definition = beanDefinitions.get(name);
    if (definition != null) return createBean(definition);

    throw new BeanNotFoundException("No bean named '" + name + "' is registered");
}
```

## 24.9 Upgrading `ClasspathScanner`

**Modifying:** `src/main/java/com/simplespringmvc/scan/ClasspathScanner.java`

The scanner changes from **instantiate + register** to **register definition + refresh**.

**Change 1:** Constructor now takes `SimpleBeanContainer` (needed for `registerBeanDefinition`).

**Change 2:** `doScan()` registers BeanDefinitions instead of instantiating:

```java
private int doScan(String basePackage) {
    Set<Class<?>> candidates = findCandidateComponents(basePackage);
    int count = 0;
    for (Class<?> clazz : candidates) {
        String beanName = deriveBeanName(clazz);
        if (!beanContainer.containsBean(beanName)) {
            beanContainer.registerBeanDefinition(beanName, clazz);  // ← was: instantiate + registerBean
            count++;
        }
    }
    return count;
}
```

**Change 3:** `scan()` calls `container.refresh()` at the end:

```java
public int scan(String... basePackages) {
    int count = 0;
    for (String basePackage : basePackages) {
        count += doScan(basePackage);
    }
    beanContainer.refresh();  // ← NEW: instantiate with DI
    return count;
}
```

**Change 4:** The `instantiate()` method is removed entirely — the container handles instantiation now.

---

## 24.10 Try It Yourself

<details><summary>Challenge 1: Implement a BeanPostProcessor that logs every bean creation</summary>

```java
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("Bean created: " + beanName + " [" + bean.getClass().getSimpleName() + "]");
        return bean;
    }
}

// Usage:
SimpleBeanContainer container = new SimpleBeanContainer();
container.addBeanPostProcessor(new LoggingBeanPostProcessor());
container.registerBeanDefinition("myService", MyService.class);
container.refresh();
// Output: Bean created: myService [MyService]
```

</details>

<details><summary>Challenge 2: Create a service with mixed constructor + field injection</summary>

```java
@Component
public class OrderService {
    // Constructor injection for required dependencies
    private final OrderRepository repository;

    // Field injection for optional/cross-cutting concerns
    @Autowired
    private NotificationService notifications;

    @Value("${order.prefix:ORD}")
    private String orderPrefix;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

    public String createOrder(String item) {
        String orderId = orderPrefix + "-" + System.currentTimeMillis();
        repository.save(orderId, item);
        notifications.notify("New order: " + orderId);
        return orderId;
    }
}
```

</details>

<details><summary>Challenge 3: Add @Qualifier to a constructor parameter</summary>

```java
@Component
public class PaymentProcessor {
    private final PaymentGateway gateway;

    // Two PaymentGateway beans exist: "stripe" and "paypal"
    public PaymentProcessor(@Qualifier("stripe") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

</details>

---

## 24.11 Tests

### Unit tests

**`ConstructorResolverTest`** — verifies constructor selection and parameter resolution:

```java
@Test
void shouldAutoDetectSingleConstructor_WhenNoAutowiredAnnotation() {
    container.registerBean("userRepository", new UserRepository());
    Object result = resolver.autowireConstructor(SingleCtorService.class);
    assertThat(((SingleCtorService) result).repository).isNotNull();
}

@Test
void shouldUseAutowiredConstructor_WhenMultipleConstructorsExist() {
    container.registerBean("userRepository", new UserRepository());
    Object result = resolver.autowireConstructor(MultiCtorService.class);
    assertThat(((MultiCtorService) result).usedAutowired).isTrue();
}

@Test
void shouldResolveValueAnnotation_WhenOnConstructorParameter() {
    container.setPropertySource(new PropertySource(Map.of("app.name", "TestApp")));
    Object result = resolver.autowireConstructor(ValueCtorService.class);
    assertThat(((ValueCtorService) result).appName).isEqualTo("TestApp");
}
```

**`AutowiredAnnotationBeanPostProcessorTest`** — verifies field/method injection:

```java
@Test
void shouldInjectField_WhenAnnotatedWithAutowired() {
    container.registerBean("orderRepository", new OrderRepository());
    var service = new OrderService();
    processor.postProcessBeforeInitialization(service, "orderService");
    assertThat(service.repository).isSameAs(container.getBean(OrderRepository.class));
}

@Test
void shouldInjectViaSetter_WhenMethodAnnotatedWithAutowired() {
    container.registerBean("orderRepository", new OrderRepository());
    var service = new SetterInjectedService();
    processor.postProcessBeforeInitialization(service, "setterInjectedService");
    assertThat(service.getRepository()).isNotNull();
}

@Test
void shouldInjectSuperclassFields_WhenParentHasAutowired() {
    container.registerBean("orderRepository", new OrderRepository());
    container.registerBean("notificationService", new NotificationService());
    var service = new ChildService();
    processor.postProcessBeforeInitialization(service, "childService");
    assertThat(service.repository).isNotNull();      // parent's field
    assertThat(service.notificationService).isNotNull(); // child's field
}
```

**`PropertySourceTest`** — verifies placeholder resolution:

```java
@Test
void shouldResolveProperty_WhenPlaceholderExists() {
    var source = new PropertySource(Map.of("server.port", "8080"));
    assertThat(source.resolveValue("${server.port}")).isEqualTo("8080");
}

@Test
void shouldUseDefault_WhenPropertyMissingAndDefaultProvided() {
    var source = new PropertySource(Map.of());
    assertThat(source.resolveValue("${server.port:8080}")).isEqualTo("8080");
}
```

### Integration test

**`ConstructorInjectionIntegrationTest`** — full lifecycle through HTTP:

```java
@Test
void shouldHandleRequest_WhenControllerWiredViaDI() throws Exception {
    SimpleBeanContainer container = new SimpleBeanContainer();
    container.registerBeanDefinition("greetingRepository", GreetingRepository.class);
    container.registerBeanDefinition("greetingController", GreetingController.class);
    container.refresh();

    SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
    tomcat = new EmbeddedTomcat(0, servlet);
    tomcat.start();

    HttpResponse<String> response = client.send(
            HttpRequest.newBuilder()
                    .uri(URI.create("http://localhost:" + tomcat.getPort() + "/greet/World"))
                    .GET().build(),
            HttpResponse.BodyHandlers.ofString());

    assertThat(response.body()).isEqualTo("Hello, World!");
}

@Test
void shouldDetectCircularDependency_WhenBeansFormCycle() {
    SimpleBeanContainer container = new SimpleBeanContainer();
    container.registerBeanDefinition("cycleA", CycleA.class);
    container.registerBeanDefinition("cycleB", CycleB.class);

    assertThatThrownBy(container::refresh)
            .isInstanceOf(CircularDependencyException.class);
}
```

---

## 24.12 Why This Works

> `★ Insight ─────────────────────────────────────`
> **Two-phase lifecycle is the key unlock.** Before ch24, registration and instantiation were the same operation (`registerBean(instance)`). Splitting them into `registerBeanDefinition(class)` + `refresh()` is what enables constructor injection — you need to know *all* classes before creating *any* instances, so you can resolve dependencies in the right order. This is the same pattern the real `AbstractApplicationContext.refresh()` follows: register all `BeanDefinition`s first, then `preInstantiateSingletons()` creates everything.
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **Single-constructor auto-detection eliminates annotation noise.** Most service classes have exactly one constructor. The real Spring (since 4.3) auto-detects this: "A class with a single constructor always uses it, even without `@Autowired`." This means `@Autowired` on a constructor is only needed when there are multiple constructors and you want to pick one. Our `ConstructorResolver.determineConstructor()` follows this same rule at line 97: `if (constructors.length == 1) return constructors[0]`.
> `─────────────────────────────────────────────────`

> `★ Insight ─────────────────────────────────────`
> **Constructor injection vs field injection: when to use which.** Constructor injection makes dependencies explicit, immutable (`final` fields), and testable without reflection. Field injection (`@Autowired` on fields) is more concise but hides dependencies. The real Spring team recommends constructor injection for required dependencies and field/setter injection only for optional ones. Our implementation mirrors the real framework by handling these in separate phases: constructor → `ConstructorResolver`, fields/methods → `AutowiredAnnotationBeanPostProcessor`.
> `─────────────────────────────────────────────────`

---

## 24.13 What We Enhanced

| Component | Before (ch01/ch17) | After (ch24) | Why |
|-----------|-------------------|--------------|-----|
| `SimpleBeanContainer` | Stores pre-instantiated singletons. `registerBean(name, instance)` only. | Two-phase lifecycle: `registerBeanDefinition(name, class)` + `refresh()`. Creates beans with constructor injection, applies BeanPostProcessors, detects circular dependencies. | Separating registration from instantiation enables dependency ordering and DI |
| `ClasspathScanner` | Instantiates beans via no-arg constructor and registers immediately | Registers BeanDefinitions (class metadata), then calls `container.refresh()` to instantiate with DI | Components can now have constructor parameters (dependencies, @Value) |
| `BeanContainer.containsBean()` | Checks only singletonMap | Also checks beanDefinitions (not yet instantiated) | Scanner must detect definitions to avoid double-registration |
| `BeanContainer.getBean(String)` | Only looks in singletonMap | Also triggers on-demand creation from definitions | Supports lazy creation during dependency resolution |

---

## 24.14 Connection to Real Framework

| Simplified | Real Spring | File:Line |
|-----------|-------------|-----------|
| `@Autowired` | `@Autowired` | `spring-beans/.../annotation/Autowired.java`:28 |
| `@Value` | `@Value` | `spring-beans/.../annotation/Value.java`:23 |
| `@Qualifier` | `@Qualifier` | `spring-beans/.../annotation/Qualifier.java`:25 |
| `BeanPostProcessor` | `BeanPostProcessor` | `spring-beans/.../config/BeanPostProcessor.java`:56 |
| `BeanDefinition` | `RootBeanDefinition` | `spring-beans/.../support/RootBeanDefinition.java` |
| `PropertySource` | `PropertySource` + `PropertySourcesPropertyResolver` | `spring-core/.../env/PropertySource.java` |
| `ConstructorResolver.determineConstructor()` | `AutowiredAnnotationBPP.determineCandidateConstructors()` | `spring-beans/.../annotation/AutowiredAnnotationBeanPostProcessor.java`:349 |
| `ConstructorResolver.autowireConstructor()` | `ConstructorResolver.autowireConstructor()` | `spring-beans/.../support/ConstructorResolver.java`:135 |
| `InjectionMetadata` | `InjectionMetadata` | `spring-beans/.../annotation/InjectionMetadata.java` |
| `InjectionMetadata.AutowiredFieldElement` | `AutowiredAnnotationBPP.AutowiredFieldElement` | `spring-beans/.../annotation/AutowiredAnnotationBeanPostProcessor.java`:722 |
| `AutowiredAnnotationBeanPostProcessor.buildInjectionMetadata()` | `AutowiredAnnotationBPP.buildAutowiringMetadata()` | `spring-beans/.../annotation/AutowiredAnnotationBeanPostProcessor.java`:547 |
| `SimpleBeanContainer.refresh()` | `AbstractApplicationContext.refresh()` | `spring-context/.../context/support/AbstractApplicationContext.java`:596 |
| `SimpleBeanContainer.createBean()` | `AbstractAutowireCapableBeanFactory.doCreateBean()` | `spring-beans/.../support/AbstractAutowireCapableBeanFactory.java`:553 |
| `SimpleBeanContainer.resolveDependency()` | `DefaultListableBeanFactory.doResolveDependency()` | `spring-beans/.../support/DefaultListableBeanFactory.java`:1556 |

> Commit: `11ab0b4351`

---

## 24.15 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/injection/Autowired.java` [NEW]

```java
package com.simplespringmvc.injection;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a constructor, field, or setter method for dependency injection.
 *
 * Maps to: {@code org.springframework.beans.factory.annotation.Autowired}
 *
 * <h3>Resolution rules:</h3>
 * <ul>
 *   <li><b>Constructor:</b> Only one constructor may be annotated with @Autowired.
 *       If a class has a single constructor, it is used automatically even without
 *       @Autowired (single-constructor auto-detection).</li>
 *   <li><b>Field:</b> Resolved by type from the container after construction.
 *       Use @Qualifier to disambiguate when multiple beans match.</li>
 *   <li><b>Method:</b> Each parameter is resolved by type from the container.
 *       Typically used on setter methods.</li>
 * </ul>
 *
 * <h3>Lifecycle placement:</h3>
 * <pre>
 *   Constructor injection → happens during bean instantiation (ConstructorResolver)
 *   Field/method injection → happens during BeanPostProcessor phase (AutowiredAnnotationBeanPostProcessor)
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No support for {@code required = false} — all dependencies are required</li>
 *   <li>No support for multiple non-required constructors (real Spring picks the greediest)</li>
 *   <li>Cannot be used on static fields or methods</li>
 * </ul>
 */
@Target({ElementType.CONSTRUCTOR, ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {
}
```

#### File: `src/main/java/com/simplespringmvc/injection/Value.java` [NEW]

```java
package com.simplespringmvc.injection;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a field or constructor parameter for injection of an externalized
 * configuration value, resolved from a {@link PropertySource}.
 *
 * Maps to: {@code org.springframework.beans.factory.annotation.Value}
 *
 * <h3>Expression syntax:</h3>
 * <ul>
 *   <li>{@code @Value("${server.port}")} — resolve property, fail if missing</li>
 *   <li>{@code @Value("${server.port:8080}")} — resolve property, use default if missing</li>
 *   <li>{@code @Value("literal")} — inject a literal string value (no ${} placeholder)</li>
 * </ul>
 *
 * <h3>Type conversion:</h3>
 * The resolved String value is automatically converted to the field's type using
 * the {@link com.simplespringmvc.convert.ConversionService} if one is registered
 * in the container. Supported types: int, long, double, float, boolean, etc.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No SpEL expression support (#{...} syntax)</li>
 *   <li>No nested placeholder resolution (${${inner}})</li>
 *   <li>Only property placeholders (${...}) and literal values</li>
 * </ul>
 */
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {

    /**
     * The value expression: a property placeholder like "${key}" or "${key:default}",
     * or a literal string value.
     */
    String value();
}
```

#### File: `src/main/java/com/simplespringmvc/injection/Qualifier.java` [NEW]

```java
package com.simplespringmvc.injection;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Disambiguates autowiring when multiple beans match the required type.
 * The qualifier value must match a bean name registered in the container.
 *
 * Maps to: {@code org.springframework.beans.factory.annotation.Qualifier}
 *
 * <h3>Usage:</h3>
 * <pre>
 *   @Autowired
 *   @Qualifier("primaryDataSource")
 *   private DataSource dataSource;
 *
 *   @Autowired
 *   public MyService(@Qualifier("cache") Store store) { ... }
 * </pre>
 *
 * <h3>Resolution order with @Qualifier:</h3>
 * <ol>
 *   <li>Find all beans matching the field/parameter type</li>
 *   <li>Filter to the bean whose name matches the qualifier value</li>
 *   <li>If no match → throw UnsatisfiedDependencyException</li>
 * </ol>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>Only matches by bean name (real Spring also supports custom qualifier annotations)</li>
 *   <li>No @Primary fallback — @Qualifier is the only disambiguation mechanism</li>
 * </ul>
 */
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Qualifier {

    /**
     * The bean name to qualify for injection.
     */
    String value();
}
```

#### File: `src/main/java/com/simplespringmvc/injection/BeanPostProcessor.java` [NEW]

```java
package com.simplespringmvc.injection;

/**
 * Factory hook that allows custom modification of new bean instances — for example,
 * checking for marker interfaces or wrapping beans with proxies.
 *
 * Maps to: {@code org.springframework.beans.factory.config.BeanPostProcessor}
 *
 * <h3>Lifecycle placement:</h3>
 * <pre>
 *   1. Constructor resolution + instantiation         (ConstructorResolver)
 *   2. postProcessBeforeInitialization()               ← you are here
 *   3. Initialization callbacks (InitializingBean, etc.) [not implemented]
 *   4. postProcessAfterInitialization()                ← and here
 * </pre>
 *
 * <h3>How this is used in Spring:</h3>
 * <ul>
 *   <li>{@code AutowiredAnnotationBeanPostProcessor} — @Autowired/@Value field/method injection
 *       (runs in the equivalent of postProcessProperties, which we map to "before")</li>
 *   <li>{@code CommonAnnotationBeanPostProcessor} — @PostConstruct, @PreDestroy, @Resource</li>
 *   <li>{@code AbstractAutoProxyCreator} — AOP proxy wrapping (runs in "after")</li>
 * </ul>
 *
 * Both methods have default implementations that return the bean unchanged,
 * so implementations only need to override the method(s) they care about.
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No PriorityOrdered/Ordered support for controlling processor order</li>
 *   <li>No SmartInstantiationAwareBeanPostProcessor (early proxy references)</li>
 *   <li>No MergedBeanDefinitionPostProcessor (metadata caching)</li>
 * </ul>
 */
public interface BeanPostProcessor {

    /**
     * Called after construction and before initialization callbacks.
     * Used for property population (e.g., @Autowired injection).
     *
     * @param bean     the new bean instance
     * @param beanName the name of the bean
     * @return the bean instance to use — either the original or a wrapped one
     */
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    /**
     * Called after initialization callbacks.
     * Used for proxy wrapping (e.g., AOP proxies).
     *
     * @param bean     the new bean instance
     * @param beanName the name of the bean
     * @return the bean instance to use — either the original or a proxy
     */
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/injection/PropertySource.java` [NEW]

```java
package com.simplespringmvc.injection;

import java.util.Collections;
import java.util.Map;

/**
 * A simple key-value property store for resolving {@link Value @Value} expressions.
 *
 * Maps to: {@code org.springframework.core.env.PropertySource} and
 * {@code org.springframework.core.env.PropertyResolver}
 *
 * <h3>Real framework hierarchy:</h3>
 * <pre>
 *   Environment
 *     └── MutablePropertySources (ordered list of PropertySource instances)
 *           ├── MapPropertySource (from application.properties)
 *           ├── SystemEnvironmentPropertySource (OS env vars)
 *           └── SystemPropertiesPropertySource (JVM -D flags)
 *   PropertySourcesPropertyResolver — resolves ${...} against all sources
 * </pre>
 *
 * We simplify to a single flat Map. The real framework layers multiple sources
 * with precedence ordering and resolves nested placeholders recursively.
 *
 * <h3>Placeholder syntax:</h3>
 * <ul>
 *   <li>{@code ${key}} — resolve, throw if missing</li>
 *   <li>{@code ${key:default}} — resolve, use default if missing</li>
 *   <li>No ${} prefix — return as literal</li>
 * </ul>
 */
public class PropertySource {

    private final Map<String, String> properties;

    public PropertySource(Map<String, String> properties) {
        this.properties = Map.copyOf(properties);
    }

    /**
     * Get a raw property value by key.
     *
     * @return the value, or null if not found
     */
    public String getProperty(String key) {
        return properties.get(key);
    }

    /**
     * Resolve a @Value expression. Handles three forms:
     * <ul>
     *   <li>{@code ${key}} — required property lookup</li>
     *   <li>{@code ${key:default}} — optional property with default</li>
     *   <li>{@code literal} — returned as-is</li>
     * </ul>
     *
     * Maps to: {@code PropertySourcesPropertyResolver.resolveRequiredPlaceholders()}
     * which delegates to {@code PropertyPlaceholderHelper.replacePlaceholders()}.
     *
     * The real framework uses a recursive helper that resolves nested placeholders
     * (${${inner.key}}). We only handle single-level placeholders.
     *
     * @param expression the @Value expression string
     * @return the resolved value
     * @throws IllegalArgumentException if a required property is missing
     */
    public String resolveValue(String expression) {
        if (!expression.startsWith("${") || !expression.endsWith("}")) {
            // Not a placeholder — return as literal
            return expression;
        }

        // Strip ${ and }
        String placeholder = expression.substring(2, expression.length() - 1);

        // Check for default value separator ':'
        int separatorIndex = placeholder.indexOf(':');
        if (separatorIndex >= 0) {
            String key = placeholder.substring(0, separatorIndex);
            String defaultValue = placeholder.substring(separatorIndex + 1);
            String value = properties.get(key);
            return value != null ? value : defaultValue;
        }

        // No default — property is required
        String value = properties.get(placeholder);
        if (value == null) {
            throw new IllegalArgumentException(
                    "Could not resolve placeholder '${" + placeholder + "}' — "
                            + "no property '" + placeholder + "' found in PropertySource");
        }
        return value;
    }

    /**
     * Return an unmodifiable view of all properties.
     */
    public Map<String, String> getAll() {
        return Collections.unmodifiableMap(properties);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/injection/BeanDefinition.java` [NEW]

```java
package com.simplespringmvc.injection;

/**
 * Metadata describing a bean to be created by the container — the class to
 * instantiate and the name to register it under.
 *
 * Maps to: {@code org.springframework.beans.factory.config.BeanDefinition}
 * (interface) and {@code org.springframework.beans.factory.support.RootBeanDefinition}
 * (primary implementation)
 *
 * <h3>Real vs simplified:</h3>
 * The real BeanDefinition is a rich metadata object storing:
 * <ul>
 *   <li>Bean class name, scope (singleton/prototype/request/session)</li>
 *   <li>Constructor argument values and property values</li>
 *   <li>Factory method, init-method, destroy-method references</li>
 *   <li>Lazy-init flag, @Primary, @DependsOn, autowire mode</li>
 *   <li>Source (annotation metadata, XML element)</li>
 * </ul>
 *
 * Our BeanDefinition holds just the bean name and class — enough to demonstrate
 * the key insight: <b>separating registration from instantiation</b> is what
 * enables dependency ordering, constructor injection, and post-processing.
 */
public class BeanDefinition {

    private final String beanName;
    private final Class<?> beanClass;

    public BeanDefinition(String beanName, Class<?> beanClass) {
        this.beanName = beanName;
        this.beanClass = beanClass;
    }

    public String getBeanName() {
        return beanName;
    }

    public Class<?> getBeanClass() {
        return beanClass;
    }

    @Override
    public String toString() {
        return "BeanDefinition{name='" + beanName + "', class=" + beanClass.getName() + "}";
    }
}
```

#### File: `src/main/java/com/simplespringmvc/injection/InjectionMetadata.java` [NEW]

```java
package com.simplespringmvc.injection;

import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.convert.ConversionService;

import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.List;

/**
 * Holds a collection of injection elements (fields and methods) discovered on a
 * target class. Orchestrates injecting all of them in order.
 *
 * Maps to: {@code org.springframework.beans.factory.annotation.InjectionMetadata}
 *
 * <h3>Real framework flow:</h3>
 * <pre>
 *   AutowiredAnnotationBeanPostProcessor.postProcessProperties()
 *     → findAutowiringMetadata(beanName, clazz)
 *       → buildAutowiringMetadata(clazz)           [walks class hierarchy]
 *         → InjectionMetadata.forElements(elements, clazz)
 *     → metadata.inject(bean, beanName, pvs)
 *       → for each InjectedElement: element.inject()
 *         → beanFactory.resolveDependency() + field.set() / method.invoke()
 * </pre>
 *
 * <h3>Inner class hierarchy:</h3>
 * <pre>
 *   InjectedElement (abstract)
 *     ├── AutowiredFieldElement   — @Autowired on a field
 *     ├── AutowiredMethodElement  — @Autowired on a setter method
 *     └── ValueFieldElement       — @Value on a field
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No PropertyValues (pvs) parameter — we don't check for explicit property definitions</li>
 *   <li>No metadata caching or needsRefresh() — rebuilt each time</li>
 *   <li>No ShortcutDependencyDescriptor for prototype optimization</li>
 * </ul>
 */
public class InjectionMetadata {

    private static final InjectionMetadata EMPTY = new InjectionMetadata(Object.class, List.of());

    private final Class<?> targetClass;
    private final List<InjectedElement> elements;

    public InjectionMetadata(Class<?> targetClass, List<InjectedElement> elements) {
        this.targetClass = targetClass;
        this.elements = List.copyOf(elements);
    }

    /**
     * Factory method — returns the EMPTY singleton when no injection points are found.
     *
     * Maps to: {@code InjectionMetadata.forElements()} (line 109)
     */
    public static InjectionMetadata forElements(List<InjectedElement> elements, Class<?> clazz) {
        return elements.isEmpty() ? EMPTY : new InjectionMetadata(clazz, elements);
    }

    /**
     * Inject all elements into the target bean.
     *
     * Maps to: {@code InjectionMetadata.inject(Object, String, PropertyValues)} (line 140)
     *
     * The real version iterates over checkedElements (filtered against the bean definition)
     * and calls element.inject(). We iterate over all elements unconditionally.
     */
    public void inject(Object target, String beanName, SimpleBeanContainer container) {
        for (InjectedElement element : elements) {
            element.inject(target, container);
        }
    }

    public Class<?> getTargetClass() {
        return targetClass;
    }

    public List<InjectedElement> getElements() {
        return elements;
    }

    // ─── Abstract base ──────────────────────────────────────────────

    /**
     * Base class for injection elements — wraps a Field or Method that needs
     * to have a value injected.
     *
     * Maps to: {@code InjectionMetadata.InjectedElement} (line 195)
     */
    public static abstract class InjectedElement {

        /**
         * Perform injection on the target instance.
         */
        public abstract void inject(Object target, SimpleBeanContainer container);
    }

    // ─── @Autowired on fields ───────────────────────────────────────

    /**
     * Injects a bean dependency into an @Autowired field.
     *
     * Maps to: {@code AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement} (line 722)
     *
     * The real version creates a DependencyDescriptor and calls
     * beanFactory.resolveDependency(). We call container.resolveDependency() directly.
     */
    public static class AutowiredFieldElement extends InjectedElement {

        private final Field field;
        private final String qualifier;

        public AutowiredFieldElement(Field field, String qualifier) {
            this.field = field;
            this.qualifier = qualifier;
        }

        @Override
        public void inject(Object target, SimpleBeanContainer container) {
            try {
                Object value = container.resolveDependency(field.getType(), qualifier);
                field.setAccessible(true);
                field.set(target, value);
            } catch (IllegalAccessException e) {
                throw new UnsatisfiedDependencyException(
                        "Failed to inject field '" + field.getName() + "' on "
                                + target.getClass().getName(), e);
            }
        }

        public Field getField() {
            return field;
        }
    }

    // ─── @Autowired on methods ──────────────────────────────────────

    /**
     * Injects bean dependencies into an @Autowired method's parameters.
     *
     * Maps to: {@code AutowiredAnnotationBeanPostProcessor.AutowiredMethodElement} (line 798)
     *
     * The real version resolves each parameter as a DependencyDescriptor,
     * then invokes the method. We resolve each parameter by type from the container.
     */
    public static class AutowiredMethodElement extends InjectedElement {

        private final Method method;

        public AutowiredMethodElement(Method method) {
            this.method = method;
        }

        @Override
        public void inject(Object target, SimpleBeanContainer container) {
            try {
                Parameter[] parameters = method.getParameters();
                Object[] args = new Object[parameters.length];

                for (int i = 0; i < parameters.length; i++) {
                    String qualifier = null;
                    Qualifier qualifierAnn = parameters[i].getAnnotation(Qualifier.class);
                    if (qualifierAnn != null) {
                        qualifier = qualifierAnn.value();
                    }
                    args[i] = container.resolveDependency(parameters[i].getType(), qualifier);
                }

                method.setAccessible(true);
                method.invoke(target, args);
            } catch (ReflectiveOperationException e) {
                throw new UnsatisfiedDependencyException(
                        "Failed to inject method '" + method.getName() + "' on "
                                + target.getClass().getName(), e);
            }
        }

        public Method getMethod() {
            return method;
        }
    }

    // ─── @Value on fields ───────────────────────────────────────────

    /**
     * Injects an externalized configuration value into a @Value field.
     *
     * Maps to: The real framework handles @Value through the same AutowiredFieldElement,
     * using a DependencyDescriptor that carries the @Value annotation. The
     * BeanFactory resolves it via AutowireCandidateResolver.getSuggestedValue()
     * → StringValueResolver → PropertyPlaceholderConfigurer.
     *
     * We simplify with a dedicated element type that resolves directly from PropertySource.
     */
    public static class ValueFieldElement extends InjectedElement {

        private final Field field;
        private final String expression;

        public ValueFieldElement(Field field, String expression) {
            this.field = field;
            this.expression = expression;
        }

        @Override
        public void inject(Object target, SimpleBeanContainer container) {
            try {
                PropertySource propertySource = container.getPropertySource();
                String resolved = propertySource.resolveValue(expression);

                // Convert to field type if needed
                Object value = convertIfNecessary(resolved, field.getType(), container);

                field.setAccessible(true);
                field.set(target, value);
            } catch (IllegalAccessException e) {
                throw new UnsatisfiedDependencyException(
                        "Failed to inject @Value field '" + field.getName() + "' on "
                                + target.getClass().getName(), e);
            }
        }

        /**
         * Convert the resolved String value to the target field type, using
         * the ConversionService if available in the container.
         */
        private Object convertIfNecessary(String value, Class<?> targetType, SimpleBeanContainer container) {
            if (targetType == String.class) {
                return value;
            }

            try {
                ConversionService conversionService = container.getBean(ConversionService.class);
                if (conversionService.canConvert(String.class, targetType)) {
                    return conversionService.convert(value, targetType);
                }
            } catch (Exception e) {
                // No ConversionService available — try basic conversions
            }

            // Fallback: basic type conversion for common types
            return basicConvert(value, targetType);
        }

        private Object basicConvert(String value, Class<?> targetType) {
            if (targetType == int.class || targetType == Integer.class) {
                return Integer.parseInt(value.trim());
            }
            if (targetType == long.class || targetType == Long.class) {
                return Long.parseLong(value.trim());
            }
            if (targetType == boolean.class || targetType == Boolean.class) {
                return Boolean.parseBoolean(value.trim());
            }
            if (targetType == double.class || targetType == Double.class) {
                return Double.parseDouble(value.trim());
            }
            throw new UnsatisfiedDependencyException(
                    "Cannot convert @Value '" + value + "' to type " + targetType.getName()
                            + " — register a ConversionService bean for advanced conversions");
        }

        public Field getField() {
            return field;
        }

        public String getExpression() {
            return expression;
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/injection/ConstructorResolver.java` [NEW]

```java
package com.simplespringmvc.injection;

import com.simplespringmvc.container.SimpleBeanContainer;

import java.lang.reflect.Constructor;
import java.lang.reflect.Parameter;

/**
 * Resolves and invokes the appropriate constructor for a bean class, autowiring
 * constructor parameters from the container.
 *
 * Maps to: {@code org.springframework.beans.factory.support.ConstructorResolver}
 * ({@code autowireConstructor()} method, line 135)
 *
 * <h3>Constructor selection algorithm:</h3>
 * <pre>
 *   1. Find @Autowired constructor → use it (error if multiple @Autowired)
 *   2. If no @Autowired and single constructor → use it (auto-detection)
 *   3. If no @Autowired and multiple constructors → use no-arg constructor
 *   4. If no suitable constructor found → throw
 * </pre>
 *
 * <h3>Real framework algorithm (simplified):</h3>
 * <pre>
 *   AutowiredAnnotationBeanPostProcessor.determineCandidateConstructors():
 *     1. Find all @Autowired constructors
 *     2. If one required=true → that's the only candidate
 *     3. If multiple required=false → all are candidates (greedy match)
 *     4. If none annotated and single constructor → auto-detect
 *
 *   ConstructorResolver.autowireConstructor():
 *     1. Sort candidates: public first, then by param count desc
 *     2. For each candidate: resolve all params via beanFactory.resolveDependency()
 *     3. Compute "type difference weight" — lowest weight wins
 *     4. Instantiate with the winning constructor + resolved args
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No greedy matching with weight scores — we take the first match</li>
 *   <li>No ConstructorProperties annotation support</li>
 *   <li>No explicit constructor argument values (XML-style)</li>
 *   <li>No caching of resolved constructors on the BeanDefinition</li>
 * </ul>
 */
public class ConstructorResolver {

    private final SimpleBeanContainer container;

    public ConstructorResolver(SimpleBeanContainer container) {
        this.container = container;
    }

    /**
     * Select the constructor, resolve its parameters from the container, and
     * create the bean instance.
     *
     * Maps to: {@code ConstructorResolver.autowireConstructor()} (line 135)
     *
     * @param beanClass the class to instantiate
     * @return the new bean instance
     */
    public Object autowireConstructor(Class<?> beanClass) {
        Constructor<?> constructor = determineConstructor(beanClass);
        Object[] args = resolveArguments(constructor);

        try {
            constructor.setAccessible(true);
            return constructor.newInstance(args);
        } catch (ReflectiveOperationException e) {
            throw new UnsatisfiedDependencyException(
                    "Failed to instantiate " + beanClass.getName()
                            + " via constructor " + constructor, e);
        }
    }

    /**
     * Select the constructor to use for instantiation.
     *
     * Maps to: {@code AutowiredAnnotationBeanPostProcessor.determineCandidateConstructors()}
     * (line 349) combined with the selection logic in
     * {@code ConstructorResolver.autowireConstructor()} (line 188)
     *
     * @param beanClass the bean class
     * @return the chosen constructor
     */
    Constructor<?> determineConstructor(Class<?> beanClass) {
        Constructor<?>[] constructors = beanClass.getDeclaredConstructors();

        // 1. Find @Autowired-annotated constructors
        Constructor<?> autowiredConstructor = null;
        int autowiredCount = 0;

        for (Constructor<?> ctor : constructors) {
            if (ctor.isAnnotationPresent(Autowired.class)) {
                autowiredConstructor = ctor;
                autowiredCount++;
            }
        }

        // Error: multiple @Autowired constructors
        if (autowiredCount > 1) {
            throw new UnsatisfiedDependencyException(
                    "Multiple @Autowired constructors found on " + beanClass.getName()
                            + ". Only one constructor may be annotated with @Autowired.");
        }

        // If an @Autowired constructor was found, use it
        if (autowiredConstructor != null) {
            return autowiredConstructor;
        }

        // 2. Single constructor auto-detection — use it regardless of params
        // This matches the real Spring behavior: "A class with a single constructor
        // always uses it, even without @Autowired"
        if (constructors.length == 1) {
            return constructors[0];
        }

        // 3. Multiple constructors — look for no-arg
        for (Constructor<?> ctor : constructors) {
            if (ctor.getParameterCount() == 0) {
                return ctor;
            }
        }

        // 4. No suitable constructor
        throw new UnsatisfiedDependencyException(
                "No suitable constructor found for " + beanClass.getName()
                        + ": no @Autowired constructor, and multiple constructors with no no-arg option.");
    }

    /**
     * Resolve each constructor parameter from the container.
     *
     * Maps to: {@code ConstructorResolver.createArgumentArray()} (line 720)
     * which creates a DependencyDescriptor per parameter and calls
     * {@code resolveAutowiredArgument()} → {@code beanFactory.resolveDependency()}.
     *
     * For each parameter:
     * <ul>
     *   <li>@Value → resolve from PropertySource</li>
     *   <li>@Qualifier → resolve bean by name</li>
     *   <li>Otherwise → resolve bean by type</li>
     * </ul>
     */
    Object[] resolveArguments(Constructor<?> constructor) {
        Parameter[] parameters = constructor.getParameters();
        Object[] args = new Object[parameters.length];

        for (int i = 0; i < parameters.length; i++) {
            args[i] = resolveParameter(parameters[i], constructor);
        }

        return args;
    }

    /**
     * Resolve a single constructor parameter.
     */
    private Object resolveParameter(Parameter parameter, Constructor<?> constructor) {
        // Check for @Value annotation — resolve from properties
        Value valueAnn = parameter.getAnnotation(Value.class);
        if (valueAnn != null) {
            return resolveValueParameter(valueAnn.value(), parameter.getType());
        }

        // Check for @Qualifier annotation — resolve bean by name
        String qualifier = null;
        Qualifier qualifierAnn = parameter.getAnnotation(Qualifier.class);
        if (qualifierAnn != null) {
            qualifier = qualifierAnn.value();
        }

        // Resolve bean dependency by type (and optional qualifier)
        try {
            return container.resolveDependency(parameter.getType(), qualifier);
        } catch (CircularDependencyException e) {
            // Let circular dependency errors propagate unmodified
            throw e;
        } catch (Exception e) {
            throw new UnsatisfiedDependencyException(
                    "Failed to resolve parameter '" + parameter.getName() + "' of type "
                            + parameter.getType().getName() + " in constructor of "
                            + constructor.getDeclaringClass().getName(), e);
        }
    }

    /**
     * Resolve a @Value parameter from PropertySource with type conversion.
     */
    private Object resolveValueParameter(String expression, Class<?> targetType) {
        PropertySource propertySource = container.getPropertySource();
        String resolved = propertySource.resolveValue(expression);

        if (targetType == String.class) {
            return resolved;
        }

        // Basic type conversion for constructor parameters
        if (targetType == int.class || targetType == Integer.class) {
            return Integer.parseInt(resolved.trim());
        }
        if (targetType == long.class || targetType == Long.class) {
            return Long.parseLong(resolved.trim());
        }
        if (targetType == boolean.class || targetType == Boolean.class) {
            return Boolean.parseBoolean(resolved.trim());
        }
        if (targetType == double.class || targetType == Double.class) {
            return Double.parseDouble(resolved.trim());
        }

        throw new UnsatisfiedDependencyException(
                "Cannot convert @Value '" + resolved + "' to type " + targetType.getName());
    }
}
```

#### File: `src/main/java/com/simplespringmvc/injection/AutowiredAnnotationBeanPostProcessor.java` [NEW]

```java
package com.simplespringmvc.injection;

import com.simplespringmvc.container.SimpleBeanContainer;

import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.ArrayList;
import java.util.List;

/**
 * The core {@link BeanPostProcessor} that handles {@link Autowired @Autowired}
 * and {@link Value @Value} injection on fields and methods.
 *
 * Maps to: {@code org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor}
 *
 * <h3>When it runs:</h3>
 * After the bean has been instantiated (constructor injection is already done),
 * this processor scans the bean's class hierarchy for @Autowired fields/methods
 * and @Value fields, resolves the dependencies, and injects them.
 *
 * <h3>Real framework flow:</h3>
 * <pre>
 *   postProcessProperties(pvs, bean, beanName)
 *     → findAutowiringMetadata(beanName, clazz, pvs)
 *       → injectionMetadataCache.get(cacheKey)          [check cache]
 *       → buildAutowiringMetadata(clazz)                 [cache miss]
 *         → walk class hierarchy (clazz → superclass → Object)
 *           → for each field: check @Autowired / @Value / @Inject
 *             → skip static fields
 *             → create AutowiredFieldElement
 *           → for each method: check @Autowired / @Inject
 *             → skip static, skip bridge methods
 *             → create AutowiredMethodElement
 *           → prepend superclass elements before subclass elements
 *       → InjectionMetadata.forElements(elements, clazz)
 *     → metadata.inject(bean, beanName, pvs)
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No metadata caching (real framework uses ConcurrentHashMap cache with double-checked locking)</li>
 *   <li>No @Inject (JSR-330) support</li>
 *   <li>No PropertyValues check for externally managed config members</li>
 *   <li>Does not process @Lookup methods</li>
 * </ul>
 */
public class AutowiredAnnotationBeanPostProcessor implements BeanPostProcessor {

    private final SimpleBeanContainer container;

    public AutowiredAnnotationBeanPostProcessor(SimpleBeanContainer container) {
        this.container = container;
    }

    /**
     * Perform field and method injection on the bean.
     *
     * Maps to: {@code postProcessProperties(PropertyValues, Object, String)} (line 490)
     *
     * In the real framework, this is actually a method from
     * {@code InstantiationAwareBeanPostProcessor} (a sub-interface of BeanPostProcessor).
     * We map it to {@code postProcessBeforeInitialization} for simplicity, since our
     * container calls "before" after construction (matching the real lifecycle timing).
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        InjectionMetadata metadata = buildInjectionMetadata(bean.getClass());
        metadata.inject(bean, beanName, container);
        return bean;
    }

    /**
     * Build injection metadata by scanning the class hierarchy for @Autowired
     * and @Value annotations on fields and methods.
     *
     * Maps to: {@code buildAutowiringMetadata(Class<?>)} (line 547)
     *
     * The real version:
     * <ol>
     *   <li>Starts from the target class, walks up to Object</li>
     *   <li>At each level, scans declared fields for @Autowired/@Value/@Inject</li>
     *   <li>Scans declared methods for @Autowired/@Inject</li>
     *   <li>Prepends parent elements before child elements (parent-first order)</li>
     *   <li>Sorts method elements via ASM for deterministic declaration order</li>
     * </ol>
     *
     * We follow the same parent-first walk but skip the ASM-based method ordering.
     */
    InjectionMetadata buildInjectionMetadata(Class<?> clazz) {
        List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();

        Class<?> targetClass = clazz;
        do {
            List<InjectionMetadata.InjectedElement> currentElements = new ArrayList<>();

            // Scan fields
            for (Field field : targetClass.getDeclaredFields()) {
                // Skip static fields — real framework: AutowiredAnnotationBPP line 574
                if (Modifier.isStatic(field.getModifiers())) {
                    continue;
                }

                if (field.isAnnotationPresent(Autowired.class)) {
                    String qualifier = null;
                    Qualifier qualifierAnn = field.getAnnotation(Qualifier.class);
                    if (qualifierAnn != null) {
                        qualifier = qualifierAnn.value();
                    }
                    currentElements.add(new InjectionMetadata.AutowiredFieldElement(field, qualifier));
                } else if (field.isAnnotationPresent(Value.class)) {
                    Value valueAnn = field.getAnnotation(Value.class);
                    currentElements.add(new InjectionMetadata.ValueFieldElement(field, valueAnn.value()));
                }
            }

            // Scan methods
            for (Method method : targetClass.getDeclaredMethods()) {
                // Skip static methods — real framework: AutowiredAnnotationBPP line 590
                if (Modifier.isStatic(method.getModifiers())) {
                    continue;
                }
                // Skip bridge methods (compiler-generated for generics)
                if (method.isBridge()) {
                    continue;
                }

                if (method.isAnnotationPresent(Autowired.class)) {
                    currentElements.add(new InjectionMetadata.AutowiredMethodElement(method));
                }
            }

            // Prepend parent elements before child — parent-first ordering
            // Real framework: line 613 — "elements.addAll(0, currElements)"
            elements.addAll(0, currentElements);

            targetClass = targetClass.getSuperclass();
        } while (targetClass != null && targetClass != Object.class);

        return InjectionMetadata.forElements(elements, clazz);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/injection/CircularDependencyException.java` [NEW]

```java
package com.simplespringmvc.injection;

/**
 * Thrown when a circular dependency is detected during bean creation.
 *
 * Maps to: {@code org.springframework.beans.factory.BeanCurrentlyInCreationException}
 *
 * <h3>Circular dependency example:</h3>
 * <pre>
 *   class A { @Autowired B b; }    ← A depends on B
 *   class B { @Autowired A a; }    ← B depends on A → CIRCULAR!
 * </pre>
 *
 * The real framework can resolve some circular dependencies for singleton beans
 * using "early references" (three-level cache in DefaultSingletonBeanRegistry):
 * <ul>
 *   <li>singletonObjects — fully initialized beans</li>
 *   <li>earlySingletonObjects — early-exposed (partially initialized)</li>
 *   <li>singletonFactories — ObjectFactory callbacks for early reference</li>
 * </ul>
 *
 * This technique only works for field/setter injection, NOT constructor injection
 * (because the object doesn't exist yet during construction). Since Spring 6.x,
 * circular dependencies require explicit @Lazy to break the cycle.
 *
 * Our simplified version detects all circular dependencies and throws immediately.
 */
public class CircularDependencyException extends RuntimeException {

    public CircularDependencyException(String message) {
        super(message);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/injection/UnsatisfiedDependencyException.java` [NEW]

```java
package com.simplespringmvc.injection;

/**
 * Thrown when a dependency cannot be resolved during autowiring — no bean of the
 * required type is registered, or a @Value property is missing.
 *
 * Maps to: {@code org.springframework.beans.factory.UnsatisfiedDependencyException}
 *
 * The real version extends BeanCreationException and includes detailed context:
 * bean name, injection point (field or parameter), dependency descriptor.
 * Ours is a simple RuntimeException with a descriptive message.
 */
public class UnsatisfiedDependencyException extends RuntimeException {

    public UnsatisfiedDependencyException(String message) {
        super(message);
    }

    public UnsatisfiedDependencyException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/container/SimpleBeanContainer.java` [MODIFIED]

See section 24.8 above for the full modified file. Key additions:
- `beanDefinitions`, `beanPostProcessors`, `currentlyCreating`, `propertySource` fields
- `registerBeanDefinition()`, `containsBeanDefinition()`, `refresh()`, `createBean()`, `resolveDependency()`
- `addBeanPostProcessor()`, `setPropertySource()`, `getPropertySource()`
- Modified `containsBean()`, `getBean(String)`, `getBean(Class)`

#### File: `src/main/java/com/simplespringmvc/scan/ClasspathScanner.java` [MODIFIED]

See section 24.9 above. Key changes:
- Constructor takes `SimpleBeanContainer` (was `BeanContainer`)
- `doScan()` calls `registerBeanDefinition()` instead of `instantiate()` + `registerBean()`
- `scan()` calls `container.refresh()` at the end
- `instantiate()` method removed

### Test Code

#### File: `src/test/java/com/simplespringmvc/injection/PropertySourceTest.java` [NEW]

10 tests covering literal values, required placeholders, defaults, immutability.

#### File: `src/test/java/com/simplespringmvc/injection/ConstructorResolverTest.java` [NEW]

10 tests covering no-arg, single-constructor auto-detection, @Autowired, @Value on params, @Qualifier, mixed params, unsatisfied deps.

#### File: `src/test/java/com/simplespringmvc/injection/AutowiredAnnotationBeanPostProcessorTest.java` [NEW]

13 tests covering @Autowired fields, @Qualifier fields, @Autowired setters, multi-arg methods, @Value fields with type conversion, mixed injection, superclass fields, static field skipping, metadata building.

#### File: `src/test/java/com/simplespringmvc/integration/ConstructorInjectionIntegrationTest.java` [NEW]

10 tests covering full DI lifecycle: constructor injection, field injection, @Value, @Qualifier, dependency ordering, circular detection, mixed registration, HTTP dispatch with DI-wired controller, @Value on constructor params, setter injection.

---

## Summary

| What | Detail |
|------|--------|
| **New classes** | `Autowired`, `Value`, `Qualifier`, `BeanPostProcessor`, `PropertySource`, `BeanDefinition`, `InjectionMetadata` (+ 3 inner classes), `ConstructorResolver`, `AutowiredAnnotationBeanPostProcessor`, `CircularDependencyException`, `UnsatisfiedDependencyException` |
| **Modified classes** | `SimpleBeanContainer` (two-phase lifecycle), `ClasspathScanner` (register definitions) |
| **New tests** | 43 tests (10 PropertySource + 10 ConstructorResolver + 13 BeanPostProcessor + 10 integration) |
| **Total tests** | 837 (all passing) |
| **Key pattern** | BeanPostProcessor — pluggable lifecycle hook between construction and use |
| **Key insight** | Separating registration from instantiation enables dependency ordering, constructor injection, and post-processing |

**Next chapter:** Feature 25 (Multipart File Upload) adds file upload support via the `MultipartResolver` strategy and Servlet 3.0 `Part` API.
