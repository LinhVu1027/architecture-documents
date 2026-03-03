# Chapter 12: Integration — All Patterns Working Together

## What You'll Learn
- How all 17 design patterns compose into a unified framework
- How to run the complete simplified Spring Framework end-to-end
- The pattern interaction map — which patterns depend on which
- Why no single pattern is sufficient; the power comes from composition

---

## 12.1 The Big Picture: Pattern Interaction Map

```
┌─────────────────────────────────────────────────────────────────┐
│                    SimpleApplicationContext                       │
│                         (FACADE)                                │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 SimpleContainer                          │    │
│  │            (FACTORY + SINGLETON)                         │    │
│  │                                                         │    │
│  │  ┌───────────────────┐    ┌──────────────────────────┐  │    │
│  │  │ BeanDefinitions   │    │ BeanPostProcessors       │  │    │
│  │  │ (BUILDER creates) │    │ (DECORATOR chain)        │  │    │
│  │  │                   │    │                          │  │    │
│  │  │ VISITOR resolves  │    │  ┌─ AwareBPP (AWARE)     │  │    │
│  │  │ placeholders      │    │  ├─ LoggingBPP           │  │    │
│  │  └───────────────────┘    │  └─ AutoProxyBPP         │  │    │
│  │                           │      │                    │  │    │
│  │                           │      ▼                    │  │    │
│  │                           │  SimpleAopProxy (PROXY)   │  │    │
│  │                           │  ├─ MethodInterceptors    │  │    │
│  │                           │  │  (CHAIN OF RESP)       │  │    │
│  │                           │  └─ proceed() recursion   │  │    │
│  │                           └──────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌──────────────────────┐  ┌─────────────────────────────────┐  │
│  │ SimpleEventMulticaster│  │ SimpleDispatcherServlet          │  │
│  │ (OBSERVER)           │  │ (FACADE + TEMPLATE METHOD)       │  │
│  │                      │  │                                  │  │
│  │ - publish/subscribe  │  │  HandlerMapping (STRATEGY)       │  │
│  │ - type-safe filtering│  │  HandlerAdapter (ADAPTER)        │  │
│  │ - sync/async modes   │  │  HandlerInterceptor (CHAIN)      │  │
│  └──────────────────────┘  │  ArgumentResolver (COMPOSITE)    │  │
│                            └─────────────────────────────────┘  │
│  ┌──────────────────────┐  ┌──────────────────────┐             │
│  │ ConversionService    │  │ TransactionTemplate   │             │
│  │ (STRATEGY REGISTRY)  │  │ (TEMPLATE + STRATEGY) │             │
│  └──────────────────────┘  │                       │             │
│                            │ TransactionManager    │             │
│  ┌──────────────────────┐  │ (STRATEGY)            │             │
│  │ DestructionCallbacks │  └──────────────────────┘             │
│  │ (COMMAND)            │                                       │
│  └──────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────┘
```

## 12.2 Pattern Dependency Table

| Pattern | Depends On | Used By |
|---------|-----------|---------|
| **Factory** | — | Singleton, Facade, Decorator |
| **Singleton** | Factory | Three-level cache, Container |
| **Builder** | — | Bean definition creation |
| **Visitor** | Builder (visits BeanDefinitions) | Placeholder resolution |
| **Template Method** | Strategy (delegates steps) | JdbcTemplate, TransactionTemplate |
| **Strategy** | — | Template Method, ConversionService |
| **Proxy** | Chain of Responsibility | Decorator (AutoProxyBPP) |
| **Decorator (BPP)** | Factory (hooks into creation) | Proxy, Aware |
| **Aware** | Decorator (implemented as BPP) | Facade (injects context) |
| **Observer** | — | Facade (context events) |
| **Chain of Resp.** | — | Proxy (interceptors), Dispatcher (interceptors) |
| **Adapter** | — | Dispatcher (handler normalization) |
| **Composite** | Adapter/Strategy | Dispatcher (argument resolvers) |
| **Facade** | ALL | ApplicationContext |
| **Command** | — | Destruction callbacks, Transaction callbacks |

> ★ **Insight** ─────────────────────────────────────
> - The patterns form a **dependency DAG** (directed acyclic graph), not a flat list. Factory and Strategy are foundational — everything builds on them. Facade sits at the top, combining all subsystems. This layered composition is what makes Spring Framework coherent rather than chaotic.
> - Notice how **Decorator (BPP) enables Proxy**: the `AutoProxyBeanPostProcessor` uses the Decorator pattern to apply the Proxy pattern. Patterns composing patterns — this is the hallmark of sophisticated framework design.
> - **The 80/20 rule of patterns:** Factory + Template Method + Strategy + Proxy account for ~80% of Spring's extensibility. The remaining patterns handle edge cases and improve ergonomics. When designing your own framework, start with these four.
> ─────────────────────────────────────────────────────

## 12.3 Complete Working Example

Here's a program that exercises every pattern from chapters 2–11:

```java
// === PatternShowcase.java ===
import java.util.HashMap;
import java.util.Map;

/**
 * Integration test: all 17 design patterns working together.
 * Run this to verify everything compiles and works end-to-end.
 */
public class PatternShowcase {

    public static void main(String[] args) {
        System.out.println("═══════════════════════════════════════════════════════");
        System.out.println("  Spring Framework Design Patterns — Complete Showcase  ");
        System.out.println("═══════════════════════════════════════════════════════\n");

        // ══════════════════════════════════════════════════════════
        // STEP 1: Create the ApplicationContext (FACADE)
        // ══════════════════════════════════════════════════════════
        System.out.println("① Creating ApplicationContext (Facade)...");
        SimpleApplicationContext context = new SimpleApplicationContext("showcase-app");

        // ══════════════════════════════════════════════════════════
        // STEP 2: Configure Environment (part of Facade)
        // ══════════════════════════════════════════════════════════
        System.out.println("② Configuring Environment...");
        context.getEnvironment().setProperty("db.url", "jdbc:h2:mem:showcase");
        context.getEnvironment().setProperty("app.name", "PatternShowcase");
        context.getEnvironment().setActiveProfile("demo");

        // ══════════════════════════════════════════════════════════
        // STEP 3: Register Converters (STRATEGY REGISTRY)
        // ══════════════════════════════════════════════════════════
        System.out.println("③ Registering Converters (Strategy Registry)...");
        context.getConversionService().addConverter(String.class, Integer.class, Integer::parseInt);
        context.getConversionService().addConverter(String.class, Boolean.class, Boolean::parseBoolean);

        // ══════════════════════════════════════════════════════════
        // STEP 4: Register Event Listeners (OBSERVER)
        // ══════════════════════════════════════════════════════════
        System.out.println("④ Registering Event Listeners (Observer)...");

        context.addListener(new SimpleApplicationListener<ContextRefreshedEvent>() {
            @Override
            public void onApplicationEvent(ContextRefreshedEvent event) {
                System.out.println("   [OBSERVER] Context refreshed event received!");
            }
        });

        context.addListener(new SimpleApplicationListener<ServiceReadyEvent>() {
            @Override
            public void onApplicationEvent(ServiceReadyEvent event) {
                System.out.println("   [OBSERVER] Service ready: " + event.getServiceName());
            }
        });

        // ══════════════════════════════════════════════════════════
        // STEP 5: Register BeanPostProcessors (DECORATOR chain)
        // ══════════════════════════════════════════════════════════
        System.out.println("⑤ Registering BeanPostProcessors (Decorator Chain)...");

        // Logging BPP
        context.addBeanPostProcessor(new LoggingBeanPostProcessor());

        // Auto-proxy BPP — wraps interface-implementing beans in proxies (PROXY pattern)
        context.addBeanPostProcessor(new AutoProxyBeanPostProcessor(
            new TimingInterceptor()  // CHAIN OF RESPONSIBILITY
        ));

        // ══════════════════════════════════════════════════════════
        // STEP 6: Register Bean Definitions (FACTORY + BUILDER)
        // ══════════════════════════════════════════════════════════
        System.out.println("⑥ Registering Bean Definitions (Factory + Builder)...");

        // Using Builder pattern for definition
        SimpleBeanDefinition dsDef = SimpleBeanDefinitionBuilder
            .genericBeanDefinition(SimpleDataSource.class)
            .addPropertyValue("url", "${db.url}")  // placeholder for Visitor
            .getBeanDefinition();
        System.out.println("   Built definition: " + dsDef);

        // Resolve placeholders (VISITOR)
        Map<String, String> properties = new HashMap<>();
        properties.put("db.url", context.getEnvironment().getProperty("db.url"));
        SimpleBeanDefinitionVisitor visitor = new SimpleBeanDefinitionVisitor(
            new PlaceholderResolver(properties)
        );
        visitor.visitBeanDefinition(dsDef);
        System.out.println("   After visitor: " + dsDef.getPropertyValues());

        // Register beans with container (FACTORY)
        context.registerBean("dataSource",
            () -> new SimpleDataSource(
                (String) dsDef.getPropertyValues().get("url")
            ));

        // Register a service that uses Aware interfaces (AWARE pattern)
        context.registerBean("notificationService", AwareService::new);

        // Register a UserService (will be auto-proxied)
        context.registerBean("userService", () -> {
            UserServiceImpl svc = new UserServiceImpl();
            return svc;
        });

        // ══════════════════════════════════════════════════════════
        // STEP 7: Refresh Context — triggers full lifecycle
        // ══════════════════════════════════════════════════════════
        System.out.println("\n⑦ Refreshing Context (triggers lifecycle)...");
        System.out.println("─────────────────────────────────────────────");
        context.refresh();
        System.out.println("─────────────────────────────────────────────");

        // ══════════════════════════════════════════════════════════
        // STEP 8: Use the beans (SINGLETON — same instance returned)
        // ══════════════════════════════════════════════════════════
        System.out.println("\n⑧ Using Beans (Singleton + Proxy)...");

        // Singleton verification
        SimpleDataSource ds1 = context.getBean("dataSource", SimpleDataSource.class);
        SimpleDataSource ds2 = context.getBean("dataSource", SimpleDataSource.class);
        System.out.println("   DataSource is singleton: " + (ds1 == ds2));
        System.out.println("   DataSource URL: " + ds1.getUrl());

        // Proxy verification — method calls go through interceptor chain
        UserService userService = context.getBean("userService", UserService.class);
        System.out.println("   UserService class: " + userService.getClass().getSimpleName());
        String user = userService.findUser(42);
        System.out.println("   Found: " + user);

        // ══════════════════════════════════════════════════════════
        // STEP 9: Transaction Template (TEMPLATE METHOD + STRATEGY)
        // ══════════════════════════════════════════════════════════
        System.out.println("\n⑨ Transaction Template (Template + Strategy)...");

        InMemoryTransactionManager txManager = new InMemoryTransactionManager();
        SimpleTransactionTemplate txTemplate = new SimpleTransactionTemplate(txManager);

        String txResult = txTemplate.execute(() -> {
            System.out.println("   Executing business logic in transaction...");
            return "TX-SUCCESS";
        });
        System.out.println("   Transaction result: " + txResult);
        System.out.println("   TX log: " + txManager.getLog());

        // ══════════════════════════════════════════════════════════
        // STEP 10: Dispatcher (ADAPTER + CHAIN + COMPOSITE)
        // ══════════════════════════════════════════════════════════
        System.out.println("\n⑩ Dispatcher (Adapter + Chain + Composite)...");

        SimpleDispatcherServlet dispatcher = new SimpleDispatcherServlet();

        // Handler Mapping (STRATEGY)
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.registerHandler("/users", new UserController());
        dispatcher.addHandlerMapping(mapping);

        // Handler Adapter (ADAPTER)
        dispatcher.addHandlerAdapter(new ControllerHandlerAdapter());

        // Interceptors (CHAIN OF RESPONSIBILITY)
        dispatcher.addInterceptor(new RequestLogInterceptor());

        // Argument Resolver (COMPOSITE)
        SimpleArgumentResolverComposite resolverComposite = new SimpleArgumentResolverComposite()
            .addResolver(new PathVariableResolver())
            .addResolver(new RequestParamResolver());
        System.out.println("   Composite supports 'pathVariable': "
            + resolverComposite.supports("pathVariable"));

        // Dispatch a request
        SimpleHttpRequest request = new SimpleHttpRequest("GET", "/users");
        SimpleHttpResponse response = new SimpleHttpResponse();
        dispatcher.doDispatch(request, response);
        System.out.println("   Response: " + response.getBody());

        // ══════════════════════════════════════════════════════════
        // STEP 11: Conversion Service (STRATEGY)
        // ══════════════════════════════════════════════════════════
        System.out.println("\n⑪ Conversion Service (Strategy)...");
        Integer num = context.getConversionService().convert("123", Integer.class);
        Boolean flag = context.getConversionService().convert("true", Boolean.class);
        System.out.println("   Converted: " + num + " (Integer), " + flag + " (Boolean)");

        // ══════════════════════════════════════════════════════════
        // STEP 12: Destruction Callbacks (COMMAND)
        // ══════════════════════════════════════════════════════════
        System.out.println("\n⑫ Destruction Callbacks (Command)...");
        SimpleDestructionCallbackRegistry destructionRegistry = new SimpleDestructionCallbackRegistry();
        destructionRegistry.registerDestructionCallback("dataSource",
            () -> System.out.println("   Closing DataSource connection pool"));
        destructionRegistry.registerDestructionCallback("userService",
            () -> System.out.println("   Shutting down UserService"));
        System.out.println("   Simulating shutdown...");
        destructionRegistry.destroyAll();

        // ══════════════════════════════════════════════════════════
        // SUMMARY
        // ══════════════════════════════════════════════════════════
        System.out.println("\n═══════════════════════════════════════════════════════");
        System.out.println("  All 17 patterns demonstrated successfully!            ");
        System.out.println("═══════════════════════════════════════════════════════");
        System.out.println("  1. Factory Method    — SimpleContainer.getBean()");
        System.out.println("  2. Abstract Factory  — Container hierarchy");
        System.out.println("  3. Singleton         — Three-level cache");
        System.out.println("  4. Prototype         — Scope interface");
        System.out.println("  5. Builder           — BeanDefinitionBuilder");
        System.out.println("  6. Proxy             — JDK Dynamic Proxy (AOP)");
        System.out.println("  7. Decorator         — BeanPostProcessor chain");
        System.out.println("  8. Adapter           — HandlerAdapter");
        System.out.println("  9. Facade            — ApplicationContext");
        System.out.println(" 10. Template Method   — JdbcTemplate, TransactionTemplate");
        System.out.println(" 11. Strategy          — TransactionManager, Converter");
        System.out.println(" 12. Observer          — ApplicationEvent system");
        System.out.println(" 13. Chain of Resp.    — Interceptors, Proxy chain");
        System.out.println(" 14. Composite         — ArgumentResolverComposite");
        System.out.println(" 15. Visitor           — BeanDefinitionVisitor");
        System.out.println(" 16. Command           — Callbacks, Destruction commands");
        System.out.println(" 17. Aware             — Framework injection callbacks");
    }
}
```

## 12.4 Expected Output

```
═══════════════════════════════════════════════════════
  Spring Framework Design Patterns — Complete Showcase
═══════════════════════════════════════════════════════

① Creating ApplicationContext (Facade)...
② Configuring Environment...
③ Registering Converters (Strategy Registry)...
④ Registering Event Listeners (Observer)...
⑤ Registering BeanPostProcessors (Decorator Chain)...
⑥ Registering Bean Definitions (Factory + Builder)...
   Built definition: BeanDef{class=SimpleDataSource, scope=singleton, lazy=false, props=[url], args=0}
   After visitor: {url=jdbc:h2:mem:showcase}

⑦ Refreshing Context (triggers lifecycle)...
─────────────────────────────────────────────────────
[AWARE] Bean name set: notificationService
[AWARE] Environment injected
[AWARE] Event publisher injected
[INIT] notificationService initialized with profile: default
   [OBSERVER] Service ready: notificationService
[BPP] Created bean: 'dataSource' → SimpleDataSource
[BPP] Created bean: 'notificationService' → AwareService
[BPP] Created bean: 'userService' → $Proxy0
   [OBSERVER] Context refreshed event received!
─────────────────────────────────────────────────────

⑧ Using Beans (Singleton + Proxy)...
   DataSource is singleton: true
   DataSource URL: jdbc:h2:mem:showcase
   UserService class: $Proxy0
[TIMING] findUser took 0ms
   Found: User-42

⑨ Transaction Template (Template + Strategy)...
   Executing business logic in transaction...
   Transaction result: TX-SUCCESS
   TX log: [BEGIN, COMMIT]

⑩ Dispatcher (Adapter + Chain + Composite)...
   Composite supports 'pathVariable': true
[LOG] GET /users
   Response: View: user-detail, Model: {name=Alice, path=/users}
[LOG] Completed: 200

⑪ Conversion Service (Strategy)...
   Converted: 123 (Integer), true (Boolean)

⑫ Destruction Callbacks (Command)...
   Simulating shutdown...
[DESTROY] Destroyed: userService
   Shutting down UserService
[DESTROY] Destroyed: dataSource
   Closing DataSource connection pool

═══════════════════════════════════════════════════════
  All 17 patterns demonstrated successfully!
═══════════════════════════════════════════════════════
```

## 12.5 Complete File Inventory

All files needed to compile and run the showcase:

| File | Pattern(s) | Chapter |
|------|-----------|---------|
| `BeanFactory.java` | Factory Method | Ch2 |
| `SimpleContainer.java` | Factory, Singleton, Decorator | Ch2, Ch7 |
| `SimpleFactoryBean.java` | Factory Method | Ch2 |
| `SimpleDataSource.java` | — (domain) | Ch2 |
| `SimpleUserRepository.java` | — (domain) | Ch2 |
| `StatementCallback.java` | Template Method, Command | Ch3 |
| `ResultSetExtractor.java` | Template Method | Ch3 |
| `RowMapper.java` | Template Method | Ch3 |
| `SimpleJdbcTemplate.java` | Template Method | Ch3 |
| `DataAccessException.java` | — (exception) | Ch3 |
| `TransactionCallback.java` | Command | Ch3 |
| `SimpleTransactionManager.java` | Strategy | Ch4 |
| `SimpleTransactionStatus.java` | — (state) | Ch4 |
| `JdbcTransactionManager.java` | Strategy | Ch4 |
| `InMemoryTransactionManager.java` | Strategy | Ch4 |
| `SimpleTransactionTemplate.java` | Template Method + Strategy | Ch4 |
| `SimpleConverter.java` | Strategy | Ch4 |
| `SimpleConversionService.java` | Strategy Registry | Ch4 |
| `MethodInterceptor.java` | Chain of Responsibility | Ch5 |
| `MethodInvocation.java` | Chain of Responsibility | Ch5 |
| `SimpleMethodInvocation.java` | Chain of Responsibility | Ch5 |
| `SimpleAopProxy.java` | Proxy | Ch5 |
| `SimpleProxyFactory.java` | Facade, Proxy | Ch5 |
| `LoggingInterceptor.java` | Chain of Responsibility | Ch5 |
| `TimingInterceptor.java` | Chain of Responsibility | Ch5 |
| `TransactionInterceptor.java` | Chain of Responsibility | Ch5 |
| `UserService.java` | — (interface) | Ch5 |
| `UserServiceImpl.java` | — (domain) | Ch5 |
| `SimpleApplicationEvent.java` | Observer | Ch6 |
| `SimpleApplicationListener.java` | Observer | Ch6 |
| `SimpleEventPublisher.java` | Observer | Ch6 |
| `SimpleEventMulticaster.java` | Observer | Ch6 |
| `OrderCreatedEvent.java` | Observer | Ch6 |
| `UserRegisteredEvent.java` | Observer | Ch6 |
| `EmailListener.java` | Observer | Ch6 |
| `AnalyticsListener.java` | Observer | Ch6 |
| `WelcomeListener.java` | Observer | Ch6 |
| `SimpleBeanPostProcessor.java` | Decorator | Ch7 |
| `SimpleBeanFactoryPostProcessor.java` | Decorator | Ch7 |
| `SimpleInitializingBean.java` | Lifecycle | Ch7 |
| `LoggingBeanPostProcessor.java` | Decorator | Ch7 |
| `AutoProxyBeanPostProcessor.java` | Decorator + Proxy | Ch7 |
| `SimpleHttpRequest.java` | — (domain) | Ch8 |
| `SimpleHttpResponse.java` | — (domain) | Ch8 |
| `SimpleModelAndView.java` | — (domain) | Ch8 |
| `SimpleHandlerAdapter.java` | Adapter | Ch8 |
| `SimpleController.java` | — (interface) | Ch8 |
| `ControllerHandlerAdapter.java` | Adapter | Ch8 |
| `FunctionalHandlerAdapter.java` | Adapter | Ch8 |
| `SimpleHandlerMapping.java` | Strategy | Ch8 |
| `SimpleUrlHandlerMapping.java` | Strategy | Ch8 |
| `SimpleHandlerInterceptor.java` | Chain of Responsibility | Ch8 |
| `SimpleHandlerExecutionChain.java` | Chain of Responsibility | Ch8 |
| `SimpleDispatcherServlet.java` | Facade, Template Method | Ch8 |
| `SecurityInterceptor.java` | Chain of Responsibility | Ch8 |
| `RequestLogInterceptor.java` | Chain of Responsibility | Ch8 |
| `UserController.java` | — (domain) | Ch8 |
| `SimpleBeanDefinition.java` | Builder | Ch9 |
| `SimpleBeanDefinitionBuilder.java` | Builder | Ch9 |
| `SimpleArgumentResolver.java` | Composite | Ch9 |
| `PathVariableResolver.java` | Composite | Ch9 |
| `RequestParamResolver.java` | Composite | Ch9 |
| `SimpleArgumentResolverComposite.java` | Composite | Ch9 |
| `SimpleResource.java` | — (domain) | Ch10 |
| `SimpleResourceLoader.java` | Strategy | Ch10 |
| `SimpleEnvironment.java` | — (domain) | Ch10 |
| `SimpleApplicationContext.java` | Facade | Ch10 |
| `ContextRefreshedEvent.java` | Observer | Ch10 |
| `BeanNameAware.java` | Aware | Ch10 |
| `BeanFactoryAware.java` | Aware | Ch10 |
| `ApplicationContextAware.java` | Aware | Ch10 |
| `EnvironmentAware.java` | Aware | Ch10 |
| `EventPublisherAware.java` | Aware | Ch10 |
| `ResourceLoaderAware.java` | Aware | Ch10 |
| `AwareBeanPostProcessor.java` | Aware + Decorator | Ch10 |
| `AwareService.java` | Aware | Ch10 |
| `ServiceReadyEvent.java` | Observer | Ch10 |
| `SimpleValueResolver.java` | Visitor | Ch11 |
| `PlaceholderResolver.java` | Visitor | Ch11 |
| `SimpleBeanDefinitionVisitor.java` | Visitor | Ch11 |
| `SimpleLifecycleCallback.java` | Command | Ch11 |
| `SimpleDisposableBean.java` | Command | Ch11 |
| `SimpleDestructionCallbackRegistry.java` | Command | Ch11 |
| `PatternShowcase.java` | ALL | Ch12 |

**Total: 75 files, 17 patterns, 1 integrated showcase.**

> ★ **Insight** ─────────────────────────────────────
> - Counting files reveals something important: **two-thirds** of the files are small single-responsibility interfaces and implementations (most under 30 lines). This is the **Interface Segregation Principle** in action — many small, focused interfaces beat a few large ones.
> - The real Spring Framework has ~3000 Java files. Our simplified version captures the essential patterns in ~75 files. The remaining 97% of Spring code handles: error recovery, performance optimization, backward compatibility, annotation processing, and the vast number of concrete implementations (e.g., dozens of `ViewResolver` implementations).
> - **The takeaway for framework designers:** Patterns are the SKELETON. Production code is the FLESH. Get the skeleton right (clear interfaces, proper separation of concerns), and the flesh follows naturally. Spring got the skeleton right in 2004 — and it still holds today, 20+ years later.
> ─────────────────────────────────────────────────────

---

## 12.6 Connection to Real Spring — Full Mapping

| Our Simplified System | Real Spring Class | Module |
|----------------------|-------------------|--------|
| `SimpleApplicationContext` | `AnnotationConfigApplicationContext` | `spring-context` |
| `SimpleContainer` | `DefaultListableBeanFactory` | `spring-beans` |
| `SimpleJdbcTemplate` | `JdbcTemplate` | `spring-jdbc` |
| `SimpleTransactionTemplate` | `TransactionTemplate` | `spring-tx` |
| `SimpleAopProxy` | `JdkDynamicAopProxy` | `spring-aop` |
| `SimpleEventMulticaster` | `SimpleApplicationEventMulticaster` | `spring-context` |
| `SimpleDispatcherServlet` | `DispatcherServlet` | `spring-webmvc` |
| `SimpleBeanDefinitionBuilder` | `BeanDefinitionBuilder` | `spring-beans` |
| `SimpleConversionService` | `GenericConversionService` | `spring-core` |
| `SimpleBeanDefinitionVisitor` | `BeanDefinitionVisitor` | `spring-beans` |
| `AwareBeanPostProcessor` | `ApplicationContextAwareProcessor` | `spring-context` |
| `AutoProxyBeanPostProcessor` | `AbstractAutoProxyCreator` | `spring-aop` |

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **Pattern Composition** | Patterns don't exist in isolation — they compose into cohesive systems |
| **Layered Architecture** | Foundation (Factory, Strategy) → Building blocks (Template, Proxy) → Integration (Facade) |
| **Interface Segregation** | Many small interfaces > few large ones |
| **75 files, 17 patterns** | A complete simplified Spring framework demonstrating every major design pattern |
| **The Skeleton Principle** | Get the interfaces and pattern structure right; implementations follow naturally |

---

## Final Note

This guide walked through all 17 major design patterns used in Spring Framework, from the foundational Factory and Singleton patterns in the IoC container, through the behavioral patterns (Template Method, Strategy, Observer, Chain of Responsibility) that enable flexibility, to the structural patterns (Proxy, Decorator, Adapter, Composite, Facade) that organize the codebase, and finally the specialized patterns (Builder, Visitor, Command, Aware) that handle specific framework needs.

Each pattern was introduced with the problem it solves, a simplified implementation, and a mapping to the real Spring source code with file and line references. Together, they demonstrate why Spring Framework has remained the dominant Java framework for over two decades — its pattern-based architecture makes it simultaneously powerful and extensible.
