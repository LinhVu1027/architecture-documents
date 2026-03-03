# Architecture: @ZaloListener Annotation Infrastructure

## Overview

The `@ZaloListener` annotation infrastructure transforms ZaloBot's listener setup from **imperative wiring** (manually creating `UpdateListener` beans and containers) into **declarative programming** (annotating methods that should handle updates). Think of it like hiring a **personal assistant** (the Bean Post-Processor) who reads your business cards (annotations), sets up phone lines (containers), and routes incoming calls (updates) to the right person (annotated methods) — all without you touching the switchboard.

This architecture mirrors Spring Kafka's `@KafkaListener` but simplifies away Kafka-specific concepts (topics, consumer groups, partitions, offsets) while preserving the core structural patterns that make the design extensible and testable.

**Module:** `zalobot-spring-boot`
**Package root:** `dev.linhvu.zalobot.boot`

---

## Interface Hierarchy

```
ZaloListenerEndpoint                              (interface)
├── getId(): String
├── getConcurrency(): Integer
├── getAutoStartup(): Boolean
├── setupListenerContainer(UpdateListenerContainer): void
│
└── AbstractZaloListenerEndpoint                  (abstract class)
    ├── id: String
    ├── concurrency: Integer
    ├── autoStartup: Boolean
    ├── setupListenerContainer(container)          ← template method
    │     └── calls createUpdateListener()
    │     └── sets listener on container
    ├── #createUpdateListener(): UpdateListener    [abstract]
    │
    └── MethodZaloListenerEndpoint                (concrete class)
        ├── bean: Object
        ├── method: Method
        └── #createUpdateListener()
              └── returns new UpdateListenerAdapter(bean, method)


ZaloListenerContainerFactory<C>                   (interface)
├── createListenerContainer(ZaloListenerEndpoint): C
│
└── ConcurrentZaloListenerContainerFactory        (concrete class)
    ├── client: ZaloBotClient
    ├── containerProperties: ContainerProperties
    ├── concurrency: int
    └── createListenerContainer(endpoint)
          ├── creates ConcurrentUpdateListenerContainer
          ├── applies endpoint configuration
          └── calls endpoint.setupListenerContainer(container)


UpdateListenerContainer                           (interface) [EXISTING]
├── setUpdateListener(UpdateListener)
├── start() / stop()
├── isRunning()
├── pause() / resume()
│
├── AbstractUpdateListenerContainer               (abstract) [EXISTING]
│   └── ConcurrentUpdateListenerContainer         (concrete) [EXISTING]
│       └── ZaloUpdateListenerContainer           (concrete) [EXISTING]

UpdateListener                                    (interface) [EXISTING]
├── onUpdate(GetUpdatesResult): void
│
└── UpdateListenerAdapter                         (concrete) [NEW]
    ├── bean: Object
    ├── method: Method
    └── onUpdate(update)
          └── invokes bean.method(update) via reflection
```

### Method Table

| Interface / Class | Key Methods | Purpose |
|---|---|---|
| `ZaloListenerEndpoint` | `getId()`, `setupListenerContainer()` | Describes a listener endpoint |
| `AbstractZaloListenerEndpoint` | `setupListenerContainer()`, `createUpdateListener()` | Template method for wiring |
| `MethodZaloListenerEndpoint` | `createUpdateListener()` | Creates adapter for annotated method |
| `ZaloListenerContainerFactory` | `createListenerContainer()` | Creates container from endpoint |
| `ConcurrentZaloListenerContainerFactory` | `createListenerContainer()` | Creates `ConcurrentUpdateListenerContainer` |
| `UpdateListenerAdapter` | `onUpdate()` | Bridges method invocation to `UpdateListener` |

---

## ASCII Class Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ANNOTATION LAYER                             │
│                                                                     │
│  ┌──────────────────┐    ┌────────────────┐                        │
│  │  @ZaloListener    │    │ @EnableZaloBot  │                        │
│  │  ───────────────  │    │ ─────────────── │                        │
│  │  id: String       │    │ @Import(        │                        │
│  │  containerFactory │    │  ZaloBootstrap  │                        │
│  │  concurrency      │    │  Configuration) │                        │
│  │  autoStartup      │    └────────────────┘                        │
│  └──────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       DISCOVERY LAYER                               │
│                                                                     │
│  ┌──────────────────────────────────────────┐                      │
│  │  ZaloListenerAnnotationBeanPostProcessor  │                      │
│  │  ─────────────────────────────────────── │                      │
│  │  - registrar: ZaloListenerEndpointRegistrar                     │
│  │  ─────────────────────────────────────── │                      │
│  │  + postProcessAfterInitialization()       │                      │
│  │  - processZaloListener(annotation, method, bean)                │
│  └──────────────────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       ENDPOINT LAYER                                │
│                                                                     │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐   │
│  │ ZaloListenerEndpoint    │    │ MethodZaloListenerEndpoint    │   │
│  │ «interface»             │◄───│                               │   │
│  │ ───────────────────     │    │ - bean: Object                │   │
│  │ + getId()               │    │ - method: Method              │   │
│  │ + getConcurrency()      │    │ ──────────────────────────── │   │
│  │ + getAutoStartup()      │    │ # createUpdateListener()     │   │
│  │ + setupListenerContainer│    └──────────────────────────────┘   │
│  └─────────────────────────┘            ▲                          │
│              ▲                          │ extends                   │
│              │ implements               │                          │
│  ┌───────────┴─────────────┐            │                          │
│  │ AbstractZaloListener    ├────────────┘                          │
│  │ Endpoint                │                                       │
│  │ ─────────────────────── │                                       │
│  │ - id: String            │                                       │
│  │ - concurrency: Integer  │                                       │
│  │ - autoStartup: Boolean  │                                       │
│  │ ─────────────────────── │                                       │
│  │ + setupListenerContainer│                                       │
│  │ # createUpdateListener  │                                       │
│  └─────────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     ADAPTER LAYER                                   │
│                                                                     │
│  ┌──────────────────────┐         ┌──────────────────────┐         │
│  │ UpdateListener       │◄────────│ UpdateListenerAdapter │         │
│  │ «interface»          │implements│                      │         │
│  │ ──────────────────── │         │ - bean: Object        │         │
│  │ + onUpdate(result)   │         │ - method: Method      │         │
│  └──────────────────────┘         │ ──────────────────── │         │
│                                   │ + onUpdate(result)    │         │
│                                   │   → bean.method(args) │         │
│                                   └──────────────────────┘         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     FACTORY LAYER                                   │
│                                                                     │
│  ┌───────────────────────────┐    ┌────────────────────────────┐   │
│  │ ZaloListenerContainer     │    │ ConcurrentZaloListener     │   │
│  │ Factory «interface»       │◄───│ ContainerFactory           │   │
│  │ ─────────────────────     │    │ ──────────────────────     │   │
│  │ + createListenerContainer │    │ - client: ZaloBotClient    │   │
│  │   (endpoint): C           │    │ - containerProperties      │   │
│  └───────────────────────────┘    │ - concurrency: int         │   │
│                                   │ ──────────────────────     │   │
│                                   │ + createListenerContainer  │   │
│                                   └────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                   REGISTRY & REGISTRAR LAYER                        │
│                                                                     │
│  ┌─────────────────────────────┐   ┌──────────────────────────┐    │
│  │ ZaloListenerEndpoint        │   │ ZaloListenerEndpoint     │    │
│  │ Registrar                   │──▶│ Registry                 │    │
│  │ ──────────────────────────  │   │ ─────────────────────    │    │
│  │ - endpointDescriptors: List │   │ - containers: Map<id,C> │    │
│  │ - endpointRegistry         │   │ ─────────────────────    │    │
│  │ ──────────────────────────  │   │ + registerContainer()   │    │
│  │ + registerEndpoint()        │   │ + getContainer(id)      │    │
│  │ + registerAllEndpoints()    │   │ + start() / stop()      │    │
│  └─────────────────────────────┘   └──────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## State Diagram

### Container Lifecycle (driven by Registry)

```
                    ┌──────────────┐
                    │   CREATED    │
                    │  (by Factory)│
                    └──────┬───────┘
                           │ registry.registerListenerContainer()
                           ▼
                    ┌──────────────┐
            ┌───── │  REGISTERED  │ ─────┐
            │      │ (in Registry)│      │
            │      └──────┬───────┘      │
            │             │              │
            │    autoStartup=true        │ autoStartup=false
            │             │              │
            │             ▼              │
            │      ┌──────────────┐      │
            │      │   STARTING   │      │
            │      └──────┬───────┘      │
            │             │              │
            │      container.start()     │
            │             │              │
            │             ▼              │
            │      ┌──────────────┐      │
            │      │   RUNNING    │◄─────┘ registry.start()
            │      │              │         (manual start)
            │      └──┬───────┬───┘
            │         │       │
            │   pause()│     │ stop()
            │         ▼       │
            │  ┌──────────┐   │
            │  │  PAUSED  │   │
            │  └──────┬───┘   │
            │         │       │
            │  resume()│      │
            │         │       │
            │         ▼       ▼
            │      ┌──────────────┐
            └─────▶│   STOPPED    │
                   └──────────────┘
```

### BPP Processing Flow

```
  Bean created by Spring
          │
          ▼
  ┌───────────────┐     No @ZaloListener
  │ BPP inspects  │────────────────────────▶ (skip, return bean)
  │ bean class     │
  └───────┬───────┘
          │ Has @ZaloListener
          ▼
  ┌───────────────┐
  │ Create Method │
  │ Endpoint      │
  └───────┬───────┘
          │
          ▼
  ┌───────────────┐
  │ Register with │
  │ Registrar     │
  └───────┬───────┘
          │
          ▼
  (return bean unchanged)
```

---

## Flows Diagram

### Flow 1: Application Startup — From @ZaloListener to Running Container

```
Step  Component                              Action
─────┬────────────────────────────────────────────────────────────────────
  1  │ @EnableZaloBot                        imports ZaloBootstrapConfiguration
     │
  2  │ ZaloBootstrapConfiguration            registers BPP bean definition
     │                                       registers Registry bean definition
     │
  3  │ Spring creates all beans              BPP.postProcessAfterInitialization()
     │                                       called for EACH bean
     │
  4  │ BPP                                   scans bean class for @ZaloListener
     │                                       on methods
     │
  5  │ BPP                                   for each @ZaloListener method:
     │   ├── creates MethodZaloListenerEndpoint
     │   ├── sets bean + method on endpoint
     │   ├── resolves id, concurrency, autoStartup
     │   └── calls registrar.registerEndpoint(endpoint, factoryName)
     │
  6  │ BPP (afterSingletonsInstantiated)     registrar.registerAllEndpoints()
     │
  7  │ Registrar                             for each stored endpoint:
     │   └── registry.registerListenerContainer(endpoint, factory)
     │
  8  │ Registry                              factory.createListenerContainer(endpoint)
     │   ├── Factory creates ConcurrentUpdateListenerContainer
     │   ├── Factory sets concurrency
     │   └── endpoint.setupListenerContainer(container)
     │       └── container.setUpdateListener(adapter)
     │
  9  │ Registry                              stores container by id
     │
 10  │ Registry (SmartLifecycle.start())     starts all autoStartup containers
     │   └── container.start()
     │       └── spawns ListenerConsumer threads
─────┴────────────────────────────────────────────────────────────────────
```

### Flow 2: Update Arrives — From API Poll to Method Invocation

```
Step  Component                              Action
─────┬────────────────────────────────────────────────────────────────────
  1  │ ZaloUpdateListenerContainer           ListenerConsumer polls Zalo API
     │   └── client.getUpdates()...call()
     │
  2  │ Zalo API                              returns ZaloApiResponse<GetUpdatesResult>
     │
  3  │ ListenerConsumer                      calls listener.onUpdate(result)
     │
  4  │ UpdateListenerAdapter                 receives onUpdate(result)
     │   ├── resolves method parameters:
     │   │   ├── GetUpdatesResult → pass directly
     │   │   ├── Message → extract result.message()
     │   │   └── String → extract result.message().text()
     │   └── invokes bean.method(resolvedArgs) via reflection
     │
  5  │ User's @ZaloListener method           executes business logic
     │   └── e.g., replies to user, logs message, etc.
─────┴────────────────────────────────────────────────────────────────────
```

---

## Design Patterns

| # | Pattern | Where Used | Why Chosen |
|---|---------|-----------|------------|
| 1 | **Adapter** | `UpdateListenerAdapter` | Bridges the gap between a reflective method call and the `UpdateListener` functional interface. Without this, users would have to implement `UpdateListener` directly instead of just annotating methods. |
| 2 | **Template Method** | `AbstractZaloListenerEndpoint.setupListenerContainer()` | Defines the skeleton of container setup (create listener → set on container) while letting subclasses (`MethodZaloListenerEndpoint`) decide HOW to create the listener. This allows future endpoint types (e.g., class-level) without changing the setup flow. |
| 3 | **Abstract Factory** | `ZaloListenerContainerFactory` | Decouples container creation from the annotation infrastructure. Users can swap in a custom factory that creates differently-configured containers without touching the BPP or registry. |
| 4 | **Registry** | `ZaloListenerEndpointRegistry` | Centralizes lifecycle management. Instead of each container managing its own start/stop, the registry manages them all — making it trivial to implement `SmartLifecycle` and integrate with Spring's application context lifecycle. |
| 5 | **Mediator** | `ZaloListenerEndpointRegistrar` | Decouples the BPP (which discovers annotations) from the registry (which manages containers). The registrar collects endpoints during bean processing and bulk-registers them after all beans are processed. |
| 6 | **Bean Post-Processor** | `ZaloListenerAnnotationBeanPostProcessor` | Spring's extension point for intercepting bean creation. This is the ONLY correct way to implement annotation-driven infrastructure that needs to process every bean in the context. |

---

## Architectural Decisions

| Decision | Chosen | Alternatives Rejected | Rationale |
|----------|--------|----------------------|-----------|
| **Where to put annotation infrastructure** | `zalobot-spring-boot` module | New `zalobot-spring` module | Project is small enough; avoid premature module splitting. Spring Kafka separates them because it supports non-Boot Spring apps, but ZaloBot targets Spring Boot primarily. |
| **Endpoint ↔ Container relationship** | Endpoint configures container via `setupListenerContainer()` | Factory configures container directly | Spring Kafka pattern; keeps endpoint self-contained. The endpoint knows best what listener to install because it holds the method reference. |
| **Method invocation mechanism** | Direct `Method.invoke()` in `UpdateListenerAdapter` | Spring's `InvocableHandlerMethod` | Simplified; avoids Spring Messaging dependency. Spring Kafka uses `InvocableHandlerMethod` for complex parameter resolution (`@Payload`, `@Header`), but ZaloBot has simpler parameter types. |
| **Parameter resolution** | Type-based matching (check parameter type at invocation time) | Annotation-based (`@Payload`, `@Header`) | ZaloBot has few parameter types (`GetUpdatesResult`, `Message`, `String`). Annotation-based resolution adds complexity without benefit for this use case. |
| **Container factory default bean name** | `"zaloListenerContainerFactory"` | Auto-detect any factory bean | Mirrors Spring Kafka convention. Explicit naming is predictable and documented. |
| **Registry as SmartLifecycle** | Yes | Separate lifecycle bean per container | Centralizes lifecycle. Registry can order startup across all containers and handle graceful shutdown coordination. |
| **BPP registers endpoints, not containers** | Endpoints registered with Registrar, Registrar feeds Registry | BPP creates containers directly | Two-phase approach allows ALL annotations to be discovered before ANY container is created. This matters when endpoints reference shared resources or need ordering. |

---

## Mapping Table: Simplified → Real Spring Kafka Source

| Simplified (ZaloBot) | Real Spring Kafka Source | File Path (in spring-kafka/) |
|---|---|---|
| `@ZaloListener` | `@KafkaListener` | `annotation/KafkaListener.java` |
| `@EnableZaloBot` | `@EnableKafka` | `annotation/EnableKafka.java` |
| `ZaloBootstrapConfiguration` | `KafkaBootstrapConfiguration` | `annotation/KafkaBootstrapConfiguration.java` |
| `ZaloListenerAnnotationBeanPostProcessor` | `KafkaListenerAnnotationBeanPostProcessor` | `annotation/KafkaListenerAnnotationBeanPostProcessor.java` |
| `ZaloListenerEndpoint` | `KafkaListenerEndpoint` | `config/KafkaListenerEndpoint.java` |
| `AbstractZaloListenerEndpoint` | `AbstractKafkaListenerEndpoint` | `config/AbstractKafkaListenerEndpoint.java` |
| `MethodZaloListenerEndpoint` | `MethodKafkaListenerEndpoint` | `config/MethodKafkaListenerEndpoint.java` |
| `ZaloListenerContainerFactory` | `KafkaListenerContainerFactory` | `config/KafkaListenerContainerFactory.java` |
| `ConcurrentZaloListenerContainerFactory` | `ConcurrentKafkaListenerContainerFactory` | `config/ConcurrentKafkaListenerContainerFactory.java` |
| `ZaloListenerEndpointRegistrar` | `KafkaListenerEndpointRegistrar` | `config/KafkaListenerEndpointRegistrar.java` |
| `ZaloListenerEndpointRegistry` | `KafkaListenerEndpointRegistry` | `config/KafkaListenerEndpointRegistry.java` |
| `UpdateListenerAdapter` | `RecordMessagingMessageListenerAdapter` | `listener/adapter/RecordMessagingMessageListenerAdapter.java` |
| `UpdateListener` (existing) | `MessageListener` / `AcknowledgingConsumerAwareMessageListener` | `listener/MessageListener.java` |
| `ConcurrentUpdateListenerContainer` (existing) | `ConcurrentMessageListenerContainer` | `listener/ConcurrentMessageListenerContainer.java` |

---

## New Classes to Create

```
zalobot-spring-boot/src/main/java/dev/linhvu/zalobot/boot/
├── annotation/
│   ├── ZaloListener.java                               [Ch02]
│   ├── EnableZaloBot.java                               [Ch08]
│   ├── ZaloBootstrapConfiguration.java                  [Ch08]
│   └── ZaloListenerAnnotationBeanPostProcessor.java     [Ch07]
├── config/
│   ├── ZaloListenerEndpoint.java                        [Ch04]
│   ├── AbstractZaloListenerEndpoint.java                [Ch04]
│   ├── MethodZaloListenerEndpoint.java                  [Ch04]
│   ├── ZaloListenerContainerFactory.java                [Ch05]
│   ├── ConcurrentZaloListenerContainerFactory.java      [Ch05]
│   ├── ZaloListenerEndpointRegistrar.java               [Ch06]
│   └── ZaloListenerEndpointRegistry.java                [Ch06]
└── listener/
    └── adapter/
        └── UpdateListenerAdapter.java                   [Ch03]
```

---

## What We Simplified Away

| Spring Kafka Feature | Why Omitted |
|---|---|
| **Topics / TopicPartitions / TopicPattern** | ZaloBot polls a single API endpoint, not multiple topics. There's no concept of topic routing. |
| **Consumer Groups** | ZaloBot doesn't have consumer group semantics. Each bot has one update stream. |
| **Offsets / Acknowledgment** | ZaloBot's long-polling API doesn't expose offset-based consumption. |
| **Batch Listeners** | Updates arrive one at a time from the Zalo API; no batch semantics. |
| **@KafkaHandler (class-level)** | Simplified to method-level only. Class-level multi-method dispatch adds significant complexity. |
| **Retry Topics** | No topic concept, so retry-to-different-topic doesn't apply. Error handling stays in `ErrorHandler`. |
| **SpEL Expressions** | Annotation attributes use plain strings resolved via property placeholders only, not full SpEL. |
| **@SendTo / Reply** | ZaloBot's reply mechanism (sendMessage API) is different from Kafka's reply-to-topic pattern. |
| **Message Converters** | ZaloBot has a fixed message format (GetUpdatesResult); no need for pluggable converters. |
| **Record Filter Strategy** | Can be added later; not in initial implementation to keep it simple. |
| **InvocableHandlerMethod** | Uses direct `Method.invoke()` instead of Spring's handler method infrastructure. Simpler parameter resolution. |
| **Custom Scope (ListenerScope)** | Spring Kafka uses this for SpEL `__listener` references; not needed without SpEL. |

---

## Chapter Roadmap

| Chapter | Title | Core Concept |
|---------|-------|-------------|
| Ch01 | The Problem | Why manual `UpdateListener` beans don't scale |
| Ch02 | Annotation Design | Defining `@ZaloListener` — the user-facing API |
| Ch03 | The Message Adapter | `UpdateListenerAdapter` — bridging methods to `UpdateListener` |
| Ch04 | The Endpoint Abstraction | `ZaloListenerEndpoint` — bundling "what to listen" with "how to handle" |
| Ch05 | The Container Factory | `ZaloListenerContainerFactory` — creating containers from endpoints |
| Ch06 | Registry & Registrar | Managing container lifecycles and endpoint collection |
| Ch07 | The Bean Post-Processor | Discovering `@ZaloListener` annotations on beans |
| Ch08 | Bootstrap & Enablement | `@EnableZaloBot` and wiring the infrastructure |
| Ch09 | Spring Boot Auto-Configuration | Making it all work with zero configuration |
