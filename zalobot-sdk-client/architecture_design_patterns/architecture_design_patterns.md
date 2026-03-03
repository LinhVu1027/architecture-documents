# Architecture: Design Patterns in Spring Framework

## Overview

Spring Framework is a masterclass in design pattern application — a **restaurant analogy** makes this tangible. Imagine a restaurant where:
- The **kitchen** (IoC Container) manages all ingredients (beans) and recipes (definitions)
- **Waiters** (Adapters) translate customer orders into kitchen instructions
- The **maître d'** (DispatcherServlet) routes guests to the right table
- **Sous-chefs** (BeanPostProcessors) decorate dishes before serving
- The **intercom** (Event System) announces when dishes are ready

Spring uses **17+ Gang of Four patterns** and several framework-specific patterns, often combining multiple patterns in a single component. This document maps every major pattern to its real source code.

---

## 1. Interface Hierarchy — IoC Container

```
                          BeanFactory  (root SPI)
                          ├── getBean(String)
                          ├── getBean(Class<T>)
                          ├── containsBean(String)
                          ├── isSingleton(String)
                          └── isPrototype(String)
                               │
              ┌────────────────┼───────────────────┐
              │                │                   │
    HierarchicalBeanFactory  ListableBeanFactory  AutowireCapableBeanFactory
    ├── getParentBeanFactory  ├── getBeanNamesForType  ├── createBean
    └── containsLocalBean    ├── getBeansOfType        ├── autowireBean
                             └── getBeanDefinitionNames└── applyBPP
              │                │                   │
              └────────────────┼───────────────────┘
                               │
                  ConfigurableListableBeanFactory
                  ├── registerSingleton
                  ├── addBeanPostProcessor
                  ├── registerScope
                  └── preInstantiateSingletons
                               │
                  DefaultListableBeanFactory (THE implementation)
                  ├── beanDefinitionMap
                  ├── singletonObjects
                  └── all creation/resolution logic
```

### ApplicationContext — The Facade

```
         EnvironmentCapable    ListableBeanFactory    HierarchicalBeanFactory
                │                     │                        │
                └─────────────────────┼────────────────────────┘
                                      │
                           ApplicationContext  ◄── THE FACADE
                           ├── getId()
                           ├── getApplicationName()
                           ├── getParent()
                           └── getAutowireCapableBeanFactory()
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
    MessageSource        ApplicationEventPublisher    ResourcePatternResolver
    ├── getMessage       ├── publishEvent              ├── getResources
```

---

## 2. ASCII Class Diagram — Core Implementations

```
┌─────────────────────────────────────────────────────────┐
│              DefaultSingletonBeanRegistry                │
│─────────────────────────────────────────────────────────│
│ - singletonObjects: Map<String, Object>                  │  ◄ 1st-level cache
│ - earlySingletonObjects: Map<String, Object>             │  ◄ 2nd-level cache
│ - singletonFactories: Map<String, ObjectFactory<?>>      │  ◄ 3rd-level cache
│ - singletonsCurrentlyInCreation: Set<String>             │
│ - singletonLock: ReentrantLock                           │
│─────────────────────────────────────────────────────────│
│ + getSingleton(name): Object                             │
│ + getSingleton(name, ObjectFactory): Object              │
│ + registerSingleton(name, object): void                  │
└────────────────────────┬────────────────────────────────┘
                         │ extends
┌────────────────────────┴────────────────────────────────┐
│                  AbstractBeanFactory                     │
│─────────────────────────────────────────────────────────│
│ - beanPostProcessors: List<BeanPostProcessor>            │
│ - scopes: Map<String, Scope>                             │
│─────────────────────────────────────────────────────────│
│ + doGetBean(name, type, args): T     ◄ TEMPLATE METHOD   │
│ + createBean(name, def, args): Object  (abstract)        │
└────────────────────────┬────────────────────────────────┘
                         │ extends
┌────────────────────────┴────────────────────────────────┐
│           AbstractAutowireCapableBeanFactory             │
│─────────────────────────────────────────────────────────│
│ + createBean(name, def, args): Object                    │
│ + doCreateBean(name, def, args): Object                  │
│ + populateBean(name, def, wrapper): void                 │
│ + initializeBean(name, bean, def): Object                │
└────────────────────────┬────────────────────────────────┘
                         │ extends
┌────────────────────────┴────────────────────────────────┐
│              DefaultListableBeanFactory                   │
│  implements ConfigurableListableBeanFactory              │
│─────────────────────────────────────────────────────────│
│ - beanDefinitionMap: Map<String, BeanDefinition>         │
│ - beanDefinitionNames: List<String>                      │
│─────────────────────────────────────────────────────────│
│ + registerBeanDefinition(name, def): void                │
│ + preInstantiateSingletons(): void                       │
│ + resolveDependency(descriptor, name): Object            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    JdbcTemplate                          │
│  extends JdbcAccessor implements JdbcOperations           │
│─────────────────────────────────────────────────────────│
│ - fetchSize: int                                         │
│ - maxRows: int                                           │
│ - queryTimeout: int                                      │
│─────────────────────────────────────────────────────────│
│ + execute(ConnectionCallback<T>): T   ◄ TEMPLATE METHOD  │
│ + execute(StatementCallback<T>): T    ◄ TEMPLATE METHOD  │
│ + query(sql, ResultSetExtractor): T   ◄ CALLBACK         │
│ + query(sql, RowMapper<T>): List<T>   ◄ CALLBACK         │
│ + update(sql, args...): int           ◄ TEMPLATE METHOD  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  DispatcherServlet                        │
│  extends FrameworkServlet                                │
│─────────────────────────────────────────────────────────│
│ - handlerMappings: List<HandlerMapping>         STRATEGY  │
│ - handlerAdapters: List<HandlerAdapter>         ADAPTER   │
│ - viewResolvers: List<ViewResolver>             STRATEGY  │
│ - handlerExceptionResolvers: List<...>          CHAIN     │
│─────────────────────────────────────────────────────────│
│ # doDispatch(request, response): void  ◄ FACADE + TMPL   │
│ - getHandler(request): HandlerExecutionChain             │
│ - getHandlerAdapter(handler): HandlerAdapter             │
│ - processDispatchResult(...): void                       │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  JdkDynamicAopProxy                      │
│  implements AopProxy, InvocationHandler                  │
│─────────────────────────────────────────────────────────│
│ - advised: AdvisedSupport                                │
│ - cache: ProxyCallCache                                  │
│─────────────────────────────────────────────────────────│
│ + getProxy(ClassLoader): Object        ◄ FACTORY METHOD  │
│ + invoke(proxy, method, args): Object  ◄ PROXY           │
│   → builds interceptor chain                             │
│   → calls ReflectiveMethodInvocation.proceed()           │
└─────────────────────────────────────────────────────────┘
```

---

## 3. State Diagram — Bean Lifecycle

```
                    ┌──────────────┐
                    │ DEFINITION   │  BeanDefinition registered
                    │  REGISTERED  │  in BeanDefinitionMap
                    └──────┬───────┘
                           │ preInstantiateSingletons()
                           ▼
                    ┌──────────────┐
                    │ INSTANTIATING│  Constructor called
                    │              │  (or FactoryBean.getObject())
                    └──────┬───────┘
                           │ populateBean()
                           ▼
                    ┌──────────────┐
                    │  POPULATING  │  Dependencies injected
                    │              │  @Autowired, @Value resolved
                    └──────┬───────┘
                           │ initializeBean()
                           ▼
            ┌──────────────────────────────┐
            │      INITIALIZING            │
            │──────────────────────────────│
            │ 1. Aware callbacks           │  BeanNameAware, BeanFactoryAware...
            │ 2. BPP.postProcessBefore...  │  BeanPostProcessor chain
            │ 3. @PostConstruct            │  InitDestroyAnnotationBPP
            │ 4. InitializingBean          │  afterPropertiesSet()
            │ 5. Custom init-method        │  <bean init-method="...">
            │ 6. BPP.postProcessAfter...   │  BeanPostProcessor chain (proxying here)
            └──────────────┬───────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   READY      │  Bean is fully initialized
                    │  (in use)    │  Available via getBean()
                    └──────┬───────┘
                           │ container shutdown
                           ▼
            ┌──────────────────────────────┐
            │      DESTROYING              │
            │──────────────────────────────│
            │ 1. @PreDestroy               │
            │ 2. DisposableBean.destroy()  │
            │ 3. Custom destroy-method     │
            └──────────────────────────────┘
```

### Singleton Three-Level Cache State

```
    getSingleton("A")
         │
         ▼
  ┌─ singletonObjects ─┐    HIT → return fully initialized bean
  │   (1st level cache) │
  └──────────┬──────────┘
             │ MISS
             ▼
  ┌─ earlySingletonObjects ─┐    HIT → return early reference (circular ref)
  │   (2nd level cache)      │
  └──────────┬───────────────┘
             │ MISS
             ▼
  ┌─ singletonFactories ─┐    HIT → invoke factory, promote to 2nd level
  │   (3rd level cache)   │
  └──────────┬────────────┘
             │ MISS
             ▼
     Create bean from scratch
     → register ObjectFactory in 3rd level
     → after full init, promote to 1st level
```

---

## 4. Flows Diagram

### Flow 1: HTTP Request Processing (DispatcherServlet)

```
   Browser
     │
     │  HTTP GET /users/42
     ▼
┌─────────────────────────────────────────────────────────────┐
│ DispatcherServlet.doDispatch()                               │
│                                                             │
│  ① checkMultipart(request)                                  │
│     │                                                       │
│  ② getHandler(request)          ◄── iterate HandlerMappings │
│     │  RequestMappingHandlerMapping → matches @GetMapping   │
│     │  returns HandlerExecutionChain(handler + interceptors)│
│     │                                                       │
│  ③ mappedHandler.applyPreHandle()   ◄── CHAIN OF RESP      │
│     │  interceptor1.preHandle() → true                      │
│     │  interceptor2.preHandle() → true                      │
│     │                                                       │
│  ④ getHandlerAdapter(handler)   ◄── iterate HandlerAdapters│
│     │  RequestMappingHandlerAdapter.supports() → true       │
│     │                                                       │
│  ⑤ ha.handle(request, response, handler)   ◄── ADAPTER     │
│     │  resolves @PathVariable, @RequestBody                 │
│     │  invokes controller method                            │
│     │  returns ModelAndView                                 │
│     │                                                       │
│  ⑥ mappedHandler.applyPostHandle()  ◄── reverse order      │
│     │                                                       │
│  ⑦ processDispatchResult()                                  │
│     │  render(mv, request, response)                        │
│     │  viewResolver.resolveViewName() → View                │
│     │  view.render(model, request, response)                │
│     │                                                       │
│  ⑧ triggerAfterCompletion()     ◄── always, even on error  │
└─────────────────────────────────────────────────────────────┘
     │
     ▼
   Browser receives HTTP response
```

### Flow 2: Bean Creation (doGetBean)

```
  getBean("userService")
     │
     ▼
┌─ doGetBean() ────────────────────────────────────────────┐
│                                                          │
│  ① transformBeanName("userService")  → canonical name    │
│     │                                                    │
│  ② getSingleton("userService")       → check caches     │
│     │  1st level hit? → return                           │
│     │  2nd level hit? → return early ref                 │
│     │  3rd level hit? → invoke factory, promote          │
│     │                                                    │
│  ③ parentBeanFactory.containsBean()? → delegate to parent│
│     │                                                    │
│  ④ getMergedBeanDefinition()         → resolve parent defs│
│     │                                                    │
│  ⑤ Process depends-on                                    │
│     │  for each dep: getBean(dep) recursively            │
│     │                                                    │
│  ⑥ Scope-based creation:                                 │
│     │                                                    │
│     ├─ SINGLETON:                                        │
│     │   getSingleton(name, () -> createBean(...))        │
│     │   → createBeanInstance()   // Constructor           │
│     │   → populateBean()        // DI                    │
│     │   → initializeBean()      // Lifecycle             │
│     │   → addSingleton()        // Cache in 1st level    │
│     │                                                    │
│     ├─ PROTOTYPE:                                        │
│     │   beforePrototypeCreation(name)                    │
│     │   createBean(name, def, args)                      │
│     │   afterPrototypeCreation(name)                     │
│     │                                                    │
│     └─ CUSTOM SCOPE:                                     │
│         scope.get(name, () -> createBean(...))           │
│                                                          │
│  ⑦ adaptBeanInstance()  → type check + conversion        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Flow 3: AOP Proxy Invocation

```
  proxy.doSomething(args)
     │
     ▼
┌─ JdkDynamicAopProxy.invoke() ──────────────────────────┐
│                                                         │
│  ① Check special methods (equals, hashCode, toString)   │
│     │                                                   │
│  ② targetSource.getTarget()    → get real object        │
│     │                                                   │
│  ③ advised.getInterceptorsAndDynamicInterceptionAdvice()│
│     │  Matches pointcuts against method                 │
│     │  Returns List<MethodInterceptor>                  │
│     │                                                   │
│  ④ chain.isEmpty()?                                     │
│     │                                                   │
│     ├─ YES: AopUtils.invokeJoinpointUsingReflection()   │
│     │       → direct method call (no overhead)          │
│     │                                                   │
│     └─ NO:  new ReflectiveMethodInvocation(             │
│                 proxy, target, method, args, chain)      │
│             invocation.proceed()                        │
│               │                                         │
│               ├─ interceptor[0].invoke(invocation)      │
│               │    e.g., @Before advice executes        │
│               │    → invocation.proceed()               │
│               ├─ interceptor[1].invoke(invocation)      │
│               │    e.g., @Around advice wraps           │
│               │    → invocation.proceed()               │
│               └─ final: invoke actual target method     │
│                                                         │
│  ⑤ Process return value                                 │
│     │  Replace target 'this' with proxy if needed       │
│     │  Handle void/primitive edge cases                 │
│                                                         │
│  ⑥ targetSource.releaseTarget()   → return to pool      │
└─────────────────────────────────────────────────────────┘
```

---

## 5. Design Patterns Catalog

| # | Pattern | Where Used | Why Chosen |
|---|---------|-----------|------------|
| 1 | **Factory Method** | `BeanFactory.getBean()`, `FactoryBean.getObject()` | Centralizes object creation; decouples client from concrete classes |
| 2 | **Abstract Factory** | `AbstractBeanFactory` → `DefaultListableBeanFactory` hierarchy | Multiple factory implementations for different bean creation strategies |
| 3 | **Singleton** | `DefaultSingletonBeanRegistry`, three-level cache | Most beans are stateless services — sharing instances saves memory and ensures consistency |
| 4 | **Prototype** | `Scope` interface, `AbstractBeanFactory.doGetBean()` | Stateful beans need independent instances per request |
| 5 | **Builder** | `BeanDefinitionBuilder` | Complex BeanDefinition construction with many optional parameters |
| 6 | **Proxy** | `JdkDynamicAopProxy`, `ObjenesisCglibAopProxy` | Transparent cross-cutting concerns (transactions, security, caching) without modifying business code |
| 7 | **Decorator** | `BeanPostProcessor` chain, `BeanFactoryPostProcessor` | Layer behavior onto beans (proxying, validation, property injection) without subclassing |
| 8 | **Adapter** | `HandlerAdapter`, `ConverterAdapter`, `ConverterFactoryAdapter` | Unify incompatible handler types (Controllers, HttpRequestHandlers) behind one interface |
| 9 | **Facade** | `ApplicationContext`, `DispatcherServlet`, `JdbcTemplate` | Hide complex subsystem interactions behind a simplified API |
| 10 | **Template Method** | `JdbcTemplate.execute()`, `TransactionTemplate.execute()`, `AbstractBeanFactory.doGetBean()` | Fixed algorithm skeleton with customizable steps — eliminates boilerplate |
| 11 | **Strategy** | `PlatformTransactionManager`, `Converter`, `ViewResolver`, `HandlerMapping` | Swap algorithms at runtime without changing clients |
| 12 | **Observer** | `ApplicationEvent` / `ApplicationListener` / `ApplicationEventMulticaster` | Decouple event producers from consumers; enable event-driven architecture |
| 13 | **Chain of Responsibility** | `HandlerInterceptor` chain, `HandlerMapping` iteration, `ViewResolver` chain | Process requests through ordered pipeline; each link can accept or pass along |
| 14 | **Composite** | `HandlerMethodArgumentResolverComposite`, `ViewResolverComposite`, `CompositeDatabasePopulator` | Treat collections of processors as a single processor |
| 15 | **Visitor** | `BeanDefinitionVisitor` | Traverse and modify bean definition structures without changing their classes |
| 16 | **Command** | `TransactionCallback`, `StatementCallback`, `ConnectionCallback` | Encapsulate operations as objects for deferred/parameterized execution |
| 17 | **Aware (Spring-specific)** | `BeanNameAware`, `ApplicationContextAware`, `EnvironmentAware` | Controlled framework object injection without tight coupling |

---

## 6. Architectural Decisions

| Decision | Chosen | Alternative Rejected | Why |
|----------|--------|---------------------|-----|
| Three-level singleton cache | ConcurrentHashMap-based 3 caches | Single map + synchronization | Resolves circular references gracefully; avoids full locking |
| Callback over subclassing | `StatementCallback`, `TransactionCallback` | Abstract template subclasses | More flexible — lambdas, anonymous classes; no class explosion |
| JDK Proxy + CGLIB dual support | Both available, auto-selected | JDK-only or CGLIB-only | JDK proxies for interfaces (lighter); CGLIB for concrete classes (broader) |
| Event multicaster abstraction | `ApplicationEventMulticaster` interface | Hard-coded observer list | Allows sync/async switching; testability; custom filtering |
| `FactoryBean` as bean-level factory | Special `&` prefix convention | Separate factory registry | Co-locates factory with product in same container; natural Spring idiom |
| Builder for BeanDefinition | Static factory methods returning builder | Telescoping constructor | BeanDefinition has 20+ optional fields; builder prevents constructor chaos |
| `Scope` as Strategy | Custom scope implementations | Hard-coded singleton/prototype | Extensible for web scopes (request, session) and custom scopes (conversation, flow) |
| Interceptor chain over AOP for MVC | `HandlerInterceptor` separate from AOP | Use AOP for everything | Interceptors are simpler, lighter weight for request/response processing |

---

## 7. Mapping Table — Simplified ↔ Real Source

| Simplified Concept | Real Source File | Line(s) |
|-------------------|-----------------|---------|
| `SimpleContainer.getBean()` | `AbstractBeanFactory.doGetBean()` | :236-400 |
| `SingletonMap` | `DefaultSingletonBeanRegistry.singletonObjects` | :86 |
| `EarlyRefMap` | `DefaultSingletonBeanRegistry.earlySingletonObjects` | :95 |
| `SimpleProxy.invoke()` | `JdkDynamicAopProxy.invoke()` | :166-255 |
| `SimpleJdbcTemplate.query()` | `JdbcTemplate.query(String, ResultSetExtractor)` | :465-492 |
| `SimpleTxTemplate.execute()` | `TransactionTemplate.execute()` | :127-152 |
| `SimpleEventBus.publish()` | `SimpleApplicationEventMulticaster.multicastEvent()` | :132-154 |
| `SimpleDispatcher.dispatch()` | `DispatcherServlet.doDispatch()` | :935-1004 |
| `SimpleBeanPostProcessor` | `BeanPostProcessor` | :74-101 |
| `SimpleConverter<S,T>` | `Converter<S,T>` | :46 |
| `SimpleScope.get()` | `Scope.get()` | :75 |
| `SimpleHandlerAdapter` | `HandlerAdapter` | :62-76 |
| `SimpleHandlerInterceptor` | `HandlerInterceptor` | :102-156 |
| `SimpleBeanDefinitionBuilder` | `BeanDefinitionBuilder` | :46-384 |
| `SimpleResource` | `Resource` | :66-224 |

---

## 8. What We Simplified Away

| Real Feature | Why Omitted |
|-------------|-------------|
| CGLIB proxying | JDK proxying demonstrates the pattern sufficiently |
| `AbstractApplicationContext.refresh()` 12-step boot | Focus on individual patterns, not full boot sequence |
| `@Conditional`, `@Profile` processing | Configuration meta-programming is a separate concern |
| SpEL (Spring Expression Language) | Orthogonal to core patterns |
| `WebFlux` reactive stack | Patterns are same; reactive adds Mono/Flux wrappers |
| `SmartInitializingSingleton`, `SmartFactoryBean` | Optimizations on top of base patterns |
| AOP pointcut expression parsing | Pattern focus is on proxy mechanism, not expression DSL |
| Annotation-based configuration (`@Component`, `@Bean`) | These compile down to BeanDefinition registration |
| Transaction propagation semantics | Strategy pattern for transactions shown; propagation is policy detail |
| `MergedBeanDefinitionPostProcessor` | Specialization of BeanPostProcessor for annotation processing |
