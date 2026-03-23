# Chapter 18: Session Management & Flash Attributes

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Each request is stateless -- handler methods receive fresh objects every time, and redirect responses lose all context | Multi-step form wizards cannot carry state across requests, and the Post/Redirect/Get pattern cannot pass success/error messages to the redirect target | Build two subsystems -- **Session Attributes** (transparently store model attributes across requests) and **Flash Attributes** (carry one-time messages across a single redirect) -- integrated through `SimpleHandlerAdapter.handle()` and `SimpleDispatcherServlet.doDispatch()` |

---

## 18.1 The Integration Points

This feature is unique: it has **two integration points** in two different files, connecting two related but independent subsystems.

### Integration Point 1: `SimpleHandlerAdapter.handle()` -- Session Attributes Lifecycle

The handler adapter wraps every handler invocation with session attribute management:

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`

```java
@Override
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response,
                           Object handler) throws Exception {
    HandlerMethod handlerMethod = (HandlerMethod) handler;

    // ch18: Step 1 -- Get or create SessionAttributesHandler for this controller
    SessionAttributesHandler sessionAttrHandler =
            getSessionAttributesHandler(handlerMethod);

    // ch18: Step 2 -- Retrieve session attributes and store as request attribute
    // so ModelAttributeArgumentResolver can find existing instances
    Map<String, Object> sessionAttrs = sessionAttrHandler.retrieveAttributes(request);
    request.setAttribute(ModelAttributeArgumentResolver.SESSION_ATTRIBUTES_KEY, sessionAttrs);

    // ch18: Step 3 -- Create per-request SessionStatus
    SimpleSessionStatus sessionStatus = new SimpleSessionStatus();
    request.setAttribute(SessionStatusArgumentResolver.SESSION_STATUS_ATTRIBUTE, sessionStatus);

    // Steps 4-6: Resolve arguments, invoke, process return value
    Object[] args = resolveArguments(handlerMethod, request);
    Object result = invokeHandlerMethod(handlerMethod, args);
    ModelAndView mv = returnValueHandlers.handleReturnValue(
            result, handlerMethod, request, response);

    // ch18: Step 7 -- Session attributes lifecycle
    if (sessionAttrHandler.hasSessionAttributes()) {
        if (sessionStatus.isComplete()) {
            // SessionStatus.setComplete() was called -- cleanup
            sessionAttrHandler.cleanupAttributes(request);
        } else {
            // Store matching model attributes back in session.
            sessionAttrHandler.storeAttributes(request, sessionAttrs);
            if (mv != null) {
                sessionAttrHandler.storeAttributes(request, mv.getModel());
            }
        }
    }

    return mv;
}
```

This maps to `RequestMappingHandlerAdapter.invokeHandlerMethod()` (at `RequestMappingHandlerAdapter.java:194`) which creates a `ModelFactory`, initializes the model (retrieves session attributes), invokes the handler, and calls `ModelFactory.updateModel()` to store or clean up session attributes.

### Integration Point 2: `SimpleDispatcherServlet.doDispatch()` -- Flash Map Lifecycle

The dispatcher servlet manages flash attributes at the beginning and end of every request:

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
    HandlerExecutionChain mappedHandler = null;
    ModelAndView mv = null;
    Exception dispatchException = null;

    // ch18: Step 0 -- Flash map lifecycle: retrieve input, create output
    FlashMap inputFlashMap = null;
    if (flashMapManager != null) {
        inputFlashMap = flashMapManager.retrieveAndUpdate(request, response);
        if (inputFlashMap != null) {
            request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE,
                    Collections.unmodifiableMap(inputFlashMap));
        }
        request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
    }

    try {
        // ... handler lookup, interceptors, adapter invocation ...
    }

    // Step 7 (ch13/ch18): View resolution, rendering, or redirect.
    if (mv != null && mv.getViewName() != null) {
        String viewName = mv.getViewName();

        // ch18: Handle "redirect:" prefix
        if (viewName.startsWith("redirect:")) {
            String redirectUrl = viewName.substring("redirect:".length());
            handleRedirect(redirectUrl, request, response);
        } else {
            // ch18: Merge input flash attributes into model for view access
            if (inputFlashMap != null && !inputFlashMap.isEmpty()) {
                mv.getModel().putAll(inputFlashMap);
            }
            render(mv, request, response);
        }
    }
}
```

This mirrors the real `DispatcherServlet.doService()` (lines 850-857) which sets up `INPUT_FLASH_MAP_ATTRIBUTE` and `OUTPUT_FLASH_MAP_ATTRIBUTE` before `doDispatch()` is called, and `processDispatchResult()` which handles the redirect prefix.

Three key design decisions in these integration points:

1. **Components communicate via request attributes** -- the adapter stores session attributes and `SessionStatus` as request attributes so argument resolvers can find them. This avoids passing extra parameters through the resolver chain.
2. **The adapter caches `SessionAttributesHandler` per controller class** -- `@SessionAttributes` metadata is extracted once and reused, matching `RequestMappingHandlerAdapter.sessionAttributesHandlerCache` (line 194).
3. **The dispatcher handles "redirect:" prefix detection** -- view names starting with `redirect:` trigger the flash map save + HTTP redirect, separating redirect logic from the handler adapter.

To make these integration points work, we need to build two subsystems: session attributes (Section 18.2) and flash attributes (Section 18.3), plus three argument resolvers (Section 18.4) and modifications to existing files (Section 18.5).

## 18.2 Session Attributes Subsystem

The session attributes subsystem manages **conversational** session state -- temporary model attributes that persist across requests until explicitly cleared.

### The `@SessionAttributes` Annotation

A type-level annotation that declares which model attributes should be stored in the HTTP session between requests.

**New file:** `src/main/java/com/simplespringmvc/annotation/SessionAttributes.java`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface SessionAttributes {

    /**
     * Names of model attributes to store in the session.
     */
    String[] value() default {};

    /**
     * Types of model attributes to store in the session.
     * Any model attribute whose type matches one of these classes
     * will be stored in the session, regardless of its name.
     */
    Class<?>[] types() default {};
}
```

### The `@SessionAttribute` Annotation

A parameter-level annotation for accessing **permanent** session attributes (like authentication tokens). This is distinct from `@SessionAttributes` -- it reads existing session state rather than managing conversational attributes.

**New file:** `src/main/java/com/simplespringmvc/annotation/SessionAttribute.java`

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface SessionAttribute {

    /** The name of the session attribute to bind. */
    String value() default "";

    /** Whether the session attribute is required. */
    boolean required() default true;
}
```

The distinction between `@SessionAttributes` (type-level, conversational, managed) and `@SessionAttribute` (parameter-level, permanent, read-only) is a common source of confusion. They are different annotations serving different purposes.

### The `SessionStatus` Interface and `SimpleSessionStatus`

A handler method signals that its session conversation is complete by calling `SessionStatus.setComplete()`. The adapter checks this flag after invocation to decide whether to clean up session attributes.

**New file:** `src/main/java/com/simplespringmvc/session/SessionStatus.java`

```java
public interface SessionStatus {

    void setComplete();

    boolean isComplete();
}
```

**New file:** `src/main/java/com/simplespringmvc/session/SimpleSessionStatus.java`

```java
public class SimpleSessionStatus implements SessionStatus {

    private boolean complete = false;

    @Override
    public void setComplete() {
        this.complete = true;
    }

    @Override
    public boolean isComplete() {
        return complete;
    }
}
```

### The `SessionAttributeStore` Interface and `DefaultSessionAttributeStore`

The Strategy pattern: decouple session attribute storage from the actual backing store. The default implementation delegates to `HttpSession`.

**New file:** `src/main/java/com/simplespringmvc/session/SessionAttributeStore.java`

```java
public interface SessionAttributeStore {

    void storeAttribute(HttpServletRequest request, String attributeName, Object attributeValue);

    Object retrieveAttribute(HttpServletRequest request, String attributeName);

    void cleanupAttribute(HttpServletRequest request, String attributeName);
}
```

**New file:** `src/main/java/com/simplespringmvc/session/DefaultSessionAttributeStore.java`

```java
public class DefaultSessionAttributeStore implements SessionAttributeStore {

    @Override
    public void storeAttribute(HttpServletRequest request, String attributeName, Object attributeValue) {
        request.getSession().setAttribute(attributeName, attributeValue);
    }

    @Override
    public Object retrieveAttribute(HttpServletRequest request, String attributeName) {
        HttpSession session = request.getSession(false);
        if (session == null) {
            return null;
        }
        return session.getAttribute(attributeName);
    }

    @Override
    public void cleanupAttribute(HttpServletRequest request, String attributeName) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            session.removeAttribute(attributeName);
        }
    }
}
```

Note the careful use of `getSession(false)` for retrieve and cleanup -- this avoids creating a new session just to discover there is nothing to retrieve or clean.

### The `SessionAttributesHandler` (Facade)

The central coordinator. Created once per controller type and cached by the adapter. It extracts `@SessionAttributes` metadata at construction time and provides methods for the full lifecycle: retrieve, store, check, and cleanup.

**New file:** `src/main/java/com/simplespringmvc/session/SessionAttributesHandler.java`

```java
public class SessionAttributesHandler {

    private final Set<String> attributeNames = new HashSet<>();
    private final Set<Class<?>> attributeTypes = new HashSet<>();
    private final Set<String> knownAttributeNames = ConcurrentHashMap.newKeySet();
    private final SessionAttributeStore sessionAttributeStore;

    public SessionAttributesHandler(Class<?> handlerType, SessionAttributeStore store) {
        SessionAttributes ann = handlerType.getAnnotation(SessionAttributes.class);
        if (ann != null) {
            Collections.addAll(this.attributeNames, ann.value());
            Collections.addAll(this.attributeTypes, ann.types());
        }
        this.knownAttributeNames.addAll(this.attributeNames);
        this.sessionAttributeStore = store;
    }

    public boolean isHandlerSessionAttribute(String attributeName, Class<?> attributeType) {
        if (this.attributeTypes.contains(attributeType)) {
            this.knownAttributeNames.add(attributeName);  // discovered via type matching
            return true;
        }
        return this.attributeNames.contains(attributeName);
    }

    public Map<String, Object> retrieveAttributes(HttpServletRequest request) {
        Map<String, Object> attributes = new HashMap<>();
        for (String name : this.knownAttributeNames) {
            Object value = this.sessionAttributeStore.retrieveAttribute(request, name);
            if (value != null) {
                attributes.put(name, value);
            }
        }
        return attributes;
    }

    public void storeAttributes(HttpServletRequest request, Map<String, ?> attributes) {
        attributes.forEach((name, value) -> {
            if (value != null && isHandlerSessionAttribute(name, value.getClass())) {
                this.sessionAttributeStore.storeAttribute(request, name, value);
            }
        });
    }

    public void cleanupAttributes(HttpServletRequest request) {
        for (String name : this.knownAttributeNames) {
            this.sessionAttributeStore.cleanupAttribute(request, name);
        }
    }
}
```

The `knownAttributeNames` set is particularly interesting. Attributes matched by name are known from construction time. But attributes matched by type are discovered dynamically -- when `isHandlerSessionAttribute()` finds a type match, it adds the attribute name to `knownAttributeNames` so future `retrieveAttributes()` calls can look it up by name. This is how type-matched attributes get tracked for later retrieval and cleanup.

## 18.3 Flash Attributes Subsystem

The flash attributes subsystem supports the Post/Redirect/Get (PRG) pattern by carrying attributes across exactly one redirect.

### The `FlashMap`

A `HashMap` subclass that carries flash attributes along with a target request path and an expiration time.

**New file:** `src/main/java/com/simplespringmvc/flash/FlashMap.java`

```java
public class FlashMap extends LinkedHashMap<String, Object> {

    private String targetRequestPath;
    private long expirationTime;

    public void setTargetRequestPath(String path) {
        this.targetRequestPath = path;
    }

    public String getTargetRequestPath() {
        return this.targetRequestPath;
    }

    public void startExpirationPeriod(int timeToLive) {
        this.expirationTime = System.currentTimeMillis() + timeToLive * 1000L;
    }

    public boolean isExpired() {
        return this.expirationTime > 0 && System.currentTimeMillis() > this.expirationTime;
    }
}
```

Two features beyond a plain `HashMap`:

1. **Target path matching** -- If `targetRequestPath` is set, the FlashMap only matches requests to that exact path. This prevents a flash map from being consumed by the wrong request (e.g., when the user has multiple tabs open).
2. **Expiration** -- Flash maps expire after a configurable timeout (default 180 seconds). This prevents orphaned maps from accumulating if the redirect target is never reached.

### The `FlashMapManager` Interface

Strategy interface with two operations: retrieve at request start, save before redirect.

**New file:** `src/main/java/com/simplespringmvc/flash/FlashMapManager.java`

```java
public interface FlashMapManager {

    FlashMap retrieveAndUpdate(HttpServletRequest request, HttpServletResponse response);

    void saveOutputFlashMap(FlashMap flashMap, HttpServletRequest request, HttpServletResponse response);
}
```

### `SessionFlashMapManager` -- The Implementation

Stores flash maps in the HTTP session as a `List<FlashMap>` under a well-known key. On each `retrieveAndUpdate()` call, it finds the matching flash map, removes expired ones, and updates the session.

**New file:** `src/main/java/com/simplespringmvc/flash/SessionFlashMapManager.java`

```java
public class SessionFlashMapManager implements FlashMapManager {

    private static final String FLASH_MAPS_SESSION_ATTRIBUTE =
            SessionFlashMapManager.class.getName() + ".FLASH_MAPS";

    private int flashMapTimeout = 180;

    @Override
    public FlashMap retrieveAndUpdate(HttpServletRequest request, HttpServletResponse response) {
        List<FlashMap> allFlashMaps = retrieveFlashMaps(request);
        if (allFlashMaps == null || allFlashMaps.isEmpty()) {
            return null;
        }

        FlashMap match = null;
        boolean mapsModified = false;

        Iterator<FlashMap> iterator = allFlashMaps.iterator();
        while (iterator.hasNext()) {
            FlashMap flashMap = iterator.next();
            if (flashMap.isExpired()) {
                iterator.remove();
                mapsModified = true;
            } else if (match == null && isFlashMapForRequest(flashMap, request)) {
                match = flashMap;
                iterator.remove();
                mapsModified = true;
            }
        }

        if (mapsModified) {
            updateFlashMaps(allFlashMaps, request);
        }

        return match;
    }

    @Override
    public void saveOutputFlashMap(FlashMap flashMap, HttpServletRequest request,
                                    HttpServletResponse response) {
        if (flashMap.isEmpty()) {
            return;
        }
        flashMap.startExpirationPeriod(this.flashMapTimeout);

        List<FlashMap> allFlashMaps = retrieveFlashMaps(request);
        if (allFlashMaps == null) {
            allFlashMaps = new ArrayList<>();
        }
        allFlashMaps.add(flashMap);
        updateFlashMaps(allFlashMaps, request);
    }

    private boolean isFlashMapForRequest(FlashMap flashMap, HttpServletRequest request) {
        String targetPath = flashMap.getTargetRequestPath();
        if (targetPath != null) {
            String requestUri = request.getRequestURI();
            if (!requestUri.equals(targetPath) && !requestUri.equals(targetPath + "/")) {
                return false;
            }
        }
        return true;
    }
}
```

The matching algorithm has two important behaviors: (1) a FlashMap with no target path matches **any** request -- the first request after the redirect consumes it, and (2) expired maps are cleaned up as a side effect of retrieval, keeping the session tidy.

### `SimpleRedirectAttributes` -- The Controller-Facing API

The bridge between handler method code and the FlashMap infrastructure. Controllers add flash attributes here, and the dispatcher copies them into the output FlashMap during redirect processing.

**New file:** `src/main/java/com/simplespringmvc/flash/SimpleRedirectAttributes.java`

```java
public class SimpleRedirectAttributes {

    private final Map<String, Object> flashAttributes = new LinkedHashMap<>();

    public SimpleRedirectAttributes addFlashAttribute(String name, Object value) {
        this.flashAttributes.put(name, value);
        return this;
    }

    public Map<String, Object> getFlashAttributes() {
        return this.flashAttributes;
    }
}
```

The real `RedirectAttributesModelMap` also supports regular redirect attributes (which become URL query parameters on the redirect). We only support flash attributes -- the more interesting mechanism because they use session-backed, one-time-use storage.

## 18.4 Argument Resolvers

Three new argument resolvers inject session-related objects into handler method parameters.

### `SessionAttributeArgumentResolver`

Resolves `@SessionAttribute`-annotated parameters by looking up the named attribute in `HttpSession`.

**New file:** `src/main/java/com/simplespringmvc/adapter/SessionAttributeArgumentResolver.java`

```java
public class SessionAttributeArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(SessionAttribute.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        SessionAttribute annotation = parameter.getParameterAnnotation(SessionAttribute.class);

        String attributeName = annotation.value();
        if (attributeName.isEmpty()) {
            attributeName = parameter.getParameterName();
        }

        HttpSession session = request.getSession(false);
        Object value = (session != null) ? session.getAttribute(attributeName) : null;

        if (value == null && annotation.required()) {
            throw new IllegalStateException(
                    "Required session attribute '" + attributeName + "' is missing.");
        }

        return value;
    }
}
```

### `SessionStatusArgumentResolver`

Resolves `SessionStatus` parameters by returning the per-request instance stored as a request attribute by the adapter.

**New file:** `src/main/java/com/simplespringmvc/adapter/SessionStatusArgumentResolver.java`

```java
public class SessionStatusArgumentResolver implements HandlerMethodArgumentResolver {

    public static final String SESSION_STATUS_ATTRIBUTE =
            SessionStatusArgumentResolver.class.getName() + ".SESSION_STATUS";

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return SessionStatus.class.isAssignableFrom(parameter.getParameterType());
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        Object sessionStatus = request.getAttribute(SESSION_STATUS_ATTRIBUTE);
        if (sessionStatus == null) {
            throw new IllegalStateException(
                    "No SessionStatus found in request.");
        }
        return sessionStatus;
    }
}
```

### `RedirectAttributesArgumentResolver`

Creates a new `SimpleRedirectAttributes` instance, stores it as a request attribute for the dispatcher to find later, and returns it to the handler method.

**New file:** `src/main/java/com/simplespringmvc/adapter/RedirectAttributesArgumentResolver.java`

```java
public class RedirectAttributesArgumentResolver implements HandlerMethodArgumentResolver {

    public static final String REDIRECT_ATTRIBUTES_ATTRIBUTE =
            RedirectAttributesArgumentResolver.class.getName() + ".REDIRECT_ATTRIBUTES";

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return SimpleRedirectAttributes.class.isAssignableFrom(parameter.getParameterType());
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        SimpleRedirectAttributes redirectAttributes = new SimpleRedirectAttributes();
        request.setAttribute(REDIRECT_ATTRIBUTES_ATTRIBUTE, redirectAttributes);
        return redirectAttributes;
    }
}
```

All three resolvers are registered in `SimpleHandlerAdapter`'s constructor **before** the existing resolvers, so they take priority:

```java
// ch18: Session-aware resolvers come FIRST
argumentResolvers.addResolver(new SessionAttributeArgumentResolver());
argumentResolvers.addResolver(new SessionStatusArgumentResolver());
argumentResolvers.addResolver(new RedirectAttributesArgumentResolver());

// existing resolvers
argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));
```

## 18.5 Modified Existing Files

### `@ModelAttribute` Annotation Enhancement

Added `value()` attribute so the attribute name can be specified explicitly. This is needed for `@SessionAttributes` integration -- the session attribute name must match the `@ModelAttribute` name for the framework to retrieve existing instances from the session instead of creating new ones.

**Modifying:** `src/main/java/com/simplespringmvc/annotation/ModelAttribute.java`
**Change:** Add `value()` attribute

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ModelAttribute {

    /**
     * The name of the model attribute to bind.
     * If empty, the name is derived from the parameter type.
     *
     * ch18 Enhancement: Added to support explicit naming for session attribute matching.
     */
    String value() default "";
}
```

### `ModelAttributeArgumentResolver` Enhancement

Added session attribute lookup before creating a new instance. If an attribute with the same name exists in the session (via `@SessionAttributes`), it is retrieved and request parameters are bound to the existing instance.

**Modifying:** `src/main/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolver.java`
**Changes:**
- Added `SESSION_ATTRIBUTES_KEY` constant for the request attribute key
- Added `getAttributeName()` method reading from `@ModelAttribute.value()` or deriving from type
- Added `findInSessionAttributes()` method for session lookup
- Modified `resolveArgument()` to check session before creating new instance

```java
public static final String SESSION_ATTRIBUTES_KEY =
        ModelAttributeArgumentResolver.class.getName() + ".SESSION_ATTRIBUTES";

@Override
public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
    Class<?> targetType = parameter.getParameterType();
    String attrName = getAttributeName(parameter);

    // ch18: Check session attributes for existing instance
    Object target = findInSessionAttributes(attrName, request);

    if (target != null) {
        // Bind request parameters to existing session instance
        dataBinder.bind(target, request);
    } else {
        // Create new instance and bind
        target = dataBinder.bind(targetType, request);
    }

    validateIfApplicable(target, parameter);
    return target;
}
```

This is how multi-step form wizards work: the same object is retrieved from the session, updated with new form values, and stored back. The controller declares `@SessionAttributes("userForm")` on the class and `@ModelAttribute("userForm")` on each step's parameter. On step 1, no session attribute exists, so a new `UserForm` is created. On step 2, the `UserForm` from step 1 is retrieved from the session, and only the new fields are bound.

## 18.6 Try It Yourself

<details>
<summary>Exercise: Trace the multi-step wizard flow</summary>

Given this controller:

```java
@Controller
@SessionAttributes("userForm")
public class WizardController {

    @RequestMapping(path = "/wizard/step1", method = "POST")
    public ModelAndView step1(@ModelAttribute("userForm") UserForm form) {
        return new ModelAndView("step1").addObject("userForm", form);
    }

    @RequestMapping(path = "/wizard/step2", method = "POST")
    public ModelAndView step2(@ModelAttribute("userForm") UserForm form) {
        return new ModelAndView("step2").addObject("userForm", form);
    }

    @RequestMapping(path = "/wizard/finish", method = "POST")
    public String finish(SessionStatus status) {
        status.setComplete();
        return "redirect:/wizard/done";
    }
}
```

Trace the following scenario:

**Request 1:** `POST /wizard/step1` with `name=Alice`
1. Where does the `SessionAttributesHandler` for this controller get created?
2. What does `retrieveAttributes()` return (first request, no session data yet)?
3. After `step1()` returns a `ModelAndView` with `userForm`, what happens in the session lifecycle block?

**Request 2:** `POST /wizard/step2` with `age=30`
4. What does `retrieveAttributes()` return now?
5. How does `ModelAttributeArgumentResolver` find the existing `UserForm`?
6. After binding `age=30`, what are the final values of `form.getName()` and `form.getAge()`?

**Request 3:** `POST /wizard/finish`
7. What does `SessionStatus.setComplete()` trigger?
8. What happens to the `userForm` in the session?

<details>
<summary>Answers</summary>

1. In `SimpleHandlerAdapter.getSessionAttributesHandler()`, via `computeIfAbsent` -- created once and cached in `sessionAttributesHandlerCache`.
2. An empty map -- no session attributes exist yet.
3. `sessionAttrHandler.hasSessionAttributes()` returns `true` (declares "userForm"). `sessionStatus.isComplete()` is `false`. The adapter calls `storeAttributes()` with the model -- `userForm` matches the declared name, so it's stored in the `HttpSession`.
4. A map containing `{"userForm": <the UserForm from step 1>}` -- retrieved from the session.
5. The adapter stores the retrieved session attributes as a request attribute. `ModelAttributeArgumentResolver.findInSessionAttributes("userForm", request)` finds the existing instance. Instead of creating a new `UserForm`, it binds `age=30` to the existing one.
6. `form.getName()` = "Alice" (from step 1), `form.getAge()` = 30 (from step 2). The same object carries state across both steps.
7. The `SimpleSessionStatus.complete` flag is set to `true`.
8. In the session lifecycle block, `sessionStatus.isComplete()` returns `true`, so `sessionAttrHandler.cleanupAttributes(request)` is called -- this removes "userForm" from the `HttpSession`.

</details>
</details>

<details>
<summary>Exercise: Trace the Post/Redirect/Get flash attributes flow</summary>

Given this controller:

```java
@Controller
public class RedirectController {

    @RequestMapping(path = "/save", method = "POST")
    public String save(SimpleRedirectAttributes redirectAttrs) {
        redirectAttrs.addFlashAttribute("message", "Saved successfully!");
        return "redirect:/list";
    }

    @RequestMapping(path = "/list", method = "GET")
    public String list() {
        return "list-view";
    }
}
```

Trace the following scenario:

**Request 1:** `POST /save`
1. What does the `FlashMapManager.retrieveAndUpdate()` return at request start?
2. How does the `RedirectAttributesArgumentResolver` make flash attributes available to the dispatcher?
3. When the handler returns `"redirect:/list"`, what happens in `handleRedirect()`?
4. What gets stored in the session?

**Request 2:** `GET /list`
5. What does `FlashMapManager.retrieveAndUpdate()` return this time?
6. Where do the flash attributes end up in the model?

<details>
<summary>Answers</summary>

1. `null` -- no flash maps exist yet (no prior redirect).
2. The resolver creates a `SimpleRedirectAttributes` instance, stores it as a request attribute under `REDIRECT_ATTRIBUTES_ATTRIBUTE`, and returns it to the handler. The handler calls `addFlashAttribute("message", "Saved successfully!")`.
3. In `handleRedirect()`: (a) the output `FlashMap` from the request attribute is retrieved, (b) flash attributes from `SimpleRedirectAttributes` are copied into it, (c) the target path "/list" is set on the FlashMap, (d) `flashMapManager.saveOutputFlashMap()` stores it in the session, (e) `response.sendRedirect("/list")` is called.
4. A `List<FlashMap>` containing one FlashMap with `{"message": "Saved successfully!"}` and target path "/list", stored under the `FLASH_MAPS` session key.
5. The FlashMap whose target path "/list" matches the current request URI. The map is consumed (removed from the session).
6. In `doDispatch()`, the input flash map is merged into the `ModelAndView`'s model: `mv.getModel().putAll(inputFlashMap)`. The "list-view" template can access `${message}`.

</details>
</details>

---

## 18.7 Tests

### Unit Tests -- Session Subsystem

**New file:** `src/test/java/com/simplespringmvc/session/SimpleSessionStatusTest.java`

```java
@Test
void shouldNotBeCompleteInitially() {
    var status = new SimpleSessionStatus();
    assertThat(status.isComplete()).isFalse();
}

@Test
void shouldBeComplete_AfterSetComplete() {
    var status = new SimpleSessionStatus();
    status.setComplete();
    assertThat(status.isComplete()).isTrue();
}
```

**New file:** `src/test/java/com/simplespringmvc/session/DefaultSessionAttributeStoreTest.java`

```java
@Test
void shouldStoreAttribute() {
    HttpServletRequest request = mock(HttpServletRequest.class);
    HttpSession session = mock(HttpSession.class);
    when(request.getSession()).thenReturn(session);

    store.storeAttribute(request, "user", "John");

    verify(session).setAttribute("user", "John");
}

@Test
void shouldReturnNull_WhenNoSession() {
    HttpServletRequest request = mock(HttpServletRequest.class);
    when(request.getSession(false)).thenReturn(null);

    Object result = store.retrieveAttribute(request, "user");

    assertThat(result).isNull();
}
```

**New file:** `src/test/java/com/simplespringmvc/session/SessionAttributesHandlerTest.java`

```java
@SessionAttributes({"user", "cart"})
static class NameBasedController {}

@SessionAttributes(types = {String.class, Integer.class})
static class TypeBasedController {}

@Test
void shouldMatchByName() {
    var handler = new SessionAttributesHandler(
            NameBasedController.class, new DefaultSessionAttributeStore());
    assertThat(handler.isHandlerSessionAttribute("user", Object.class)).isTrue();
    assertThat(handler.isHandlerSessionAttribute("unknown", Object.class)).isFalse();
}

@Test
void shouldTrackTypeMatchedNames() {
    var handler = new SessionAttributesHandler(
            TypeBasedController.class, new DefaultSessionAttributeStore());
    handler.isHandlerSessionAttribute("myString", String.class);
    assertThat(handler.getKnownAttributeNames()).contains("myString");
}

@Test
void shouldStoreMatchingAttributes() {
    SessionAttributeStore store = mock(SessionAttributeStore.class);
    var handler = new SessionAttributesHandler(NameBasedController.class, store);

    Map<String, Object> model = Map.of("user", "John", "cart", 42, "unrelated", "skip");
    handler.storeAttributes(request, model);

    verify(store).storeAttribute(request, "user", "John");
    verify(store).storeAttribute(request, "cart", 42);
    verify(store, never()).storeAttribute(eq(request), eq("unrelated"), any());
}
```

### Unit Tests -- Flash Subsystem

**New file:** `src/test/java/com/simplespringmvc/flash/FlashMapTest.java`

```java
@Test
void shouldStoreAndRetrieve() {
    FlashMap flashMap = new FlashMap();
    flashMap.put("message", "Success!");
    assertThat(flashMap.get("message")).isEqualTo("Success!");
}

@Test
void shouldNotBeExpiredInitially() {
    FlashMap flashMap = new FlashMap();
    assertThat(flashMap.isExpired()).isFalse();
}

@Test
void shouldBeExpired_WhenTtlIsZero() throws InterruptedException {
    FlashMap flashMap = new FlashMap();
    flashMap.startExpirationPeriod(0);
    Thread.sleep(5);
    assertThat(flashMap.isExpired()).isTrue();
}
```

**New file:** `src/test/java/com/simplespringmvc/flash/SessionFlashMapManagerTest.java`

```java
@Test
void shouldSaveFlashMap() {
    FlashMap flashMap = new FlashMap();
    flashMap.put("message", "Saved!");
    flashMap.setTargetRequestPath("/target");

    when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(null);

    manager.saveOutputFlashMap(flashMap, request, response);

    verify(session).setAttribute(eq(FLASH_MAPS_KEY), argThat(arg -> {
        List<FlashMap> maps = (List<FlashMap>) arg;
        return maps.size() == 1 && maps.get(0).containsKey("message");
    }));
}

@Test
void shouldReturnMatchingFlashMap() {
    FlashMap flashMap = new FlashMap();
    flashMap.put("message", "Hello!");
    flashMap.setTargetRequestPath("/target");
    flashMap.startExpirationPeriod(180);
    List<FlashMap> maps = new ArrayList<>();
    maps.add(flashMap);

    when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(maps);
    when(request.getRequestURI()).thenReturn("/target");

    FlashMap result = manager.retrieveAndUpdate(request, response);

    assertThat(result).isNotNull();
    assertThat(result.get("message")).isEqualTo("Hello!");
}

@Test
void shouldNotReturnWrongPath() {
    FlashMap flashMap = new FlashMap();
    flashMap.setTargetRequestPath("/other");
    flashMap.startExpirationPeriod(180);
    List<FlashMap> maps = new ArrayList<>();
    maps.add(flashMap);

    when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(maps);
    when(request.getRequestURI()).thenReturn("/target");

    assertThat(manager.retrieveAndUpdate(request, response)).isNull();
}
```

**New file:** `src/test/java/com/simplespringmvc/flash/SimpleRedirectAttributesTest.java`

```java
@Test
void shouldStoreFlashAttributes() {
    var attrs = new SimpleRedirectAttributes();
    attrs.addFlashAttribute("message", "Saved!");
    attrs.addFlashAttribute("count", 3);

    assertThat(attrs.getFlashAttributes())
            .containsEntry("message", "Saved!")
            .containsEntry("count", 3);
}

@Test
void shouldSupportFluentChaining() {
    var attrs = new SimpleRedirectAttributes()
            .addFlashAttribute("a", 1)
            .addFlashAttribute("b", 2);

    assertThat(attrs.getFlashAttributes()).hasSize(2);
}
```

### Unit Tests -- Argument Resolvers

**New file:** `src/test/java/com/simplespringmvc/adapter/SessionAttributeArgumentResolverTest.java`

```java
@Test
void shouldResolveByExplicitName() throws Exception {
    // @SessionAttribute("currentUser") String user
    HttpSession session = mock(HttpSession.class);
    when(request.getSession(false)).thenReturn(session);
    when(session.getAttribute("currentUser")).thenReturn("John");

    Object result = resolver.resolveArgument(param, request);
    assertThat(result).isEqualTo("John");
}

@Test
void shouldThrow_WhenRequiredAttributeMissing() throws Exception {
    when(request.getSession(false)).thenReturn(null);

    assertThatThrownBy(() -> resolver.resolveArgument(param, request))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("currentUser");
}

@Test
void shouldReturnNull_WhenOptionalAttributeMissing() throws Exception {
    // @SessionAttribute(value = "token", required = false)
    when(request.getSession(false)).thenReturn(null);

    Object result = resolver.resolveArgument(optionalParam, request);
    assertThat(result).isNull();
}
```

**New file:** `src/test/java/com/simplespringmvc/adapter/SessionStatusArgumentResolverTest.java`

```java
@Test
void shouldResolveFromRequestAttribute() throws Exception {
    SimpleSessionStatus status = new SimpleSessionStatus();
    when(request.getAttribute(SessionStatusArgumentResolver.SESSION_STATUS_ATTRIBUTE))
            .thenReturn(status);

    Object result = resolver.resolveArgument(param, request);
    assertThat(result).isSameAs(status);
}

@Test
void shouldThrow_WhenSessionStatusNotFound() throws Exception {
    when(request.getAttribute(SessionStatusArgumentResolver.SESSION_STATUS_ATTRIBUTE))
            .thenReturn(null);

    assertThatThrownBy(() -> resolver.resolveArgument(param, request))
            .isInstanceOf(IllegalStateException.class);
}
```

**New file:** `src/test/java/com/simplespringmvc/adapter/RedirectAttributesArgumentResolverTest.java`

```java
@Test
void shouldCreateAndStoreInRequest() throws Exception {
    Object result = resolver.resolveArgument(param, request);

    assertThat(result).isInstanceOf(SimpleRedirectAttributes.class);
    verify(request).setAttribute(
            eq(RedirectAttributesArgumentResolver.REDIRECT_ATTRIBUTES_ATTRIBUTE),
            same(result));
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/SessionManagementIntegrationTest.java`

```java
@Test
void shouldStoreModelAttributeInSession() throws Exception {
    HttpServletRequest request = mockRequest("POST", "/wizard/step1");
    when(request.getParameterNames()).thenReturn(
            Collections.enumeration(List.of("name")));
    when(request.getParameter("name")).thenReturn("Alice");

    ((Servlet) servlet).service(request, response);

    verify(session).setAttribute(eq("userForm"), argThat(arg ->
            arg instanceof UserForm form && "Alice".equals(form.getName())));
}

@Test
void shouldRetrieveFromSession() throws Exception {
    UserForm existingForm = new UserForm();
    existingForm.setName("Alice");
    when(session.getAttribute("userForm")).thenReturn(existingForm);

    HttpServletRequest request = mockRequest("POST", "/wizard/step2");
    when(request.getParameterNames()).thenReturn(
            Collections.enumeration(List.of("age")));
    when(request.getParameter("age")).thenReturn("30");

    ((Servlet) servlet).service(request, response);

    assertThat(existingForm.getName()).isEqualTo("Alice");
    assertThat(existingForm.getAge()).isEqualTo(30);
}

@Test
void shouldCleanupOnComplete() throws Exception {
    HttpServletRequest request = mockRequest("POST", "/wizard/finish");
    when(request.getParameterNames()).thenReturn(Collections.emptyEnumeration());

    ((Servlet) servlet).service(request, response);

    verify(session).removeAttribute("userForm");
}

@Test
void shouldRedirectAndSaveFlashAttributes() throws Exception {
    HttpServletRequest request = mockRequest("POST", "/save");
    when(request.getParameterNames()).thenReturn(Collections.emptyEnumeration());

    ((Servlet) servlet).service(request, response);

    verify(response).sendRedirect("/list");
    verify(session).setAttribute(
            eq(SessionFlashMapManager.class.getName() + ".FLASH_MAPS"),
            argThat(arg -> {
                var maps = (List<FlashMap>) arg;
                FlashMap map = maps.get(0);
                return "Saved successfully!".equals(map.get("message"))
                        && "/list".equals(map.getTargetRequestPath());
            }));
}
```

**Run:** `./gradlew test` -- expected: all tests pass (including all prior tests)

---

## 18.8 Why This Works

> **Insight** -------------------------------------------
> **Two subsystems, two integration points, one communication mechanism.** Session attributes integrate through the handler adapter (which wraps handler invocation with retrieve/store/cleanup), while flash attributes integrate through the dispatcher servlet (which manages the flash map lifecycle around the entire dispatch). Both subsystems communicate between components using **request attributes** -- the adapter stores session data as request attributes for resolvers to find, and the dispatcher stores the output FlashMap as a request attribute for the redirect handler to retrieve. This avoids threading additional parameters through every method signature and matches the real framework's pattern of using request attributes as a per-request data bus.
> -----------------------------------------------------------

> **Insight** -------------------------------------------
> **The `knownAttributeNames` set solves the type-matching discovery problem.** When `@SessionAttributes(types = UserForm.class)` is declared, the framework doesn't know which attribute *names* to look up in the session until it has seen a `UserForm` instance at least once. The solution: `isHandlerSessionAttribute()` records the name of each type-matched attribute in `knownAttributeNames`. On subsequent requests, `retrieveAttributes()` iterates `knownAttributeNames` to retrieve them by name. This dynamic discovery mechanism is present in the real framework at `SessionAttributesHandler.java:52` and is the reason the set uses `ConcurrentHashMap.newKeySet()` -- it may be modified from multiple request threads.
> -----------------------------------------------------------

> **Insight** -------------------------------------------
> **Flash attributes are a Data Transfer Object with a built-in lifecycle.** A `FlashMap` is not just a `HashMap` -- it carries metadata (target path, expiration time) that governs its matching and cleanup. The target path ensures the flash map is consumed by the intended redirect target (not some other tab's request). The expiration time ensures orphaned maps are cleaned up even if the redirect target is never reached. This "data + metadata" pattern transforms a simple value object into a self-managing entity with built-in lifecycle rules.
> -----------------------------------------------------------

> **Insight** -------------------------------------------
> **The "redirect:" prefix is not a view name -- it is a protocol.** When the dispatcher detects `redirect:` at the start of a view name, it does not resolve a view. Instead, it switches to a completely different code path: save flash maps, send HTTP 302. This prefix-based protocol is a convention that avoids introducing a separate return type. The real framework supports both `redirect:` and `forward:` prefixes, with `RedirectView` and `InternalResourceView` as the corresponding `View` implementations. Our simplified approach detects the prefix directly in `doDispatch()`, which is less flexible but clearly shows the branching logic.
> -----------------------------------------------------------

## 18.9 What We Enhanced

| Aspect | Before (ch17) | Current (ch18) | Real Framework |
|--------|---------------|----------------|----------------|
| **Session attribute management** | None -- each request gets fresh objects | `@SessionAttributes` transparently stores/retrieves model attributes across requests | `SessionAttributesHandler` + `ModelFactory.initModel()/updateModel()` at `RequestMappingHandlerAdapter.java:194` |
| **Session attribute access** | Direct `HttpSession.getAttribute()` only | `@SessionAttribute` parameter annotation for declarative injection | `SessionAttributeMethodArgumentResolver` with `AbstractNamedValueMethodArgumentResolver` base class |
| **Session completion** | No concept of session lifecycle | `SessionStatus.setComplete()` triggers cleanup of managed attributes | `SimpleSessionStatus` / `SessionStatus` at `SessionStatus.java:33` |
| **Redirect support** | No redirect handling | `redirect:` prefix in view name triggers HTTP 302 + flash attribute save | `DispatcherServlet.processDispatchResult()` + `RedirectView` |
| **Flash attributes** | No cross-redirect data transfer | `SimpleRedirectAttributes` + `FlashMap` + `SessionFlashMapManager` carry attributes across exactly one redirect | `RedirectAttributesModelMap` + `FlashMap` + `AbstractFlashMapManager` + `SessionFlashMapManager` |
| **Model attribute naming** | Derived from parameter type only | `@ModelAttribute("name")` supports explicit naming for session attribute matching | `ModelAttribute.value()` / `name()` aliased via `@AliasFor` |

## 18.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `@SessionAttributes` | `@SessionAttributes` | `SessionAttributes.java:62` | Real version has `@AliasFor` between `value()` and `names()` |
| `SessionAttributesHandler` | `SessionAttributesHandler` | `SessionAttributesHandler.java:52` | Real version uses `WebRequest` instead of `HttpServletRequest`, tracks known attributes across distributed sessions |
| `SessionStatus` / `SimpleSessionStatus` | `SessionStatus` / `SimpleSessionStatus` | `SessionStatus.java:33` | Identical interface; real version is in `web.bind.support` package |
| `SessionAttributeStore` / `DefaultSessionAttributeStore` | `SessionAttributeStore` / `DefaultSessionAttributeStore` | `DefaultSessionAttributeStore.java:35` | Real version supports configurable attribute name prefix, uses `WebRequest.SCOPE_SESSION` |
| `FlashMap` | `FlashMap` | `FlashMap.java:51` | Real version extends `HashMap` (not `LinkedHashMap`), implements `Comparable<FlashMap>` for priority sorting, supports target request parameter matching |
| `FlashMapManager` | `FlashMapManager` | `FlashMapManager.java:31` | Identical interface contract |
| `SessionFlashMapManager` | `SessionFlashMapManager` extends `AbstractFlashMapManager` | `SessionFlashMapManager.java:36` | Real version has a base class with template methods, session mutex synchronization, URL path normalization |
| `SimpleRedirectAttributes` | `RedirectAttributesModelMap` | `RedirectAttributesModelMap.java:37` | Real version extends `ModelMap`, implements `RedirectAttributes` interface, supports regular redirect attributes (URL query params) and `DataBinder`-based value formatting |
| `SimpleHandlerAdapter.handle()` session lifecycle | `RequestMappingHandlerAdapter.invokeHandlerMethod()` + `ModelFactory` | `RequestMappingHandlerAdapter.java:130` | Real version delegates to `ModelFactory.initModel()` and `updateModel()`, uses `ModelAndViewContainer` for attribute passing |
| `SimpleDispatcherServlet.doDispatch()` flash lifecycle | `DispatcherServlet.doService()` + `doDispatch()` | `DispatcherServlet.java:221,228,853` | Real version sets up flash maps in `doService()` before `doDispatch()`, handles `redirect:` via `RedirectView` instantiation in `createDefaultViewName()` |
| `SessionAttributeArgumentResolver` | `SessionAttributeMethodArgumentResolver` | `spring-webmvc` | Real version extends `AbstractNamedValueMethodArgumentResolver`, supports type conversion and `Optional` parameters |
| `RedirectAttributesArgumentResolver` | `RedirectAttributesMethodArgumentResolver` | `spring-webmvc` | Real version creates `RedirectAttributesModelMap` with `DataBinder` for value formatting, sets redirect model on `ModelAndViewContainer` |

## 18.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/annotation/SessionAttributes.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Type-level annotation that declares which model attributes should be
 * transparently stored in the HTTP session between requests.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.SessionAttributes}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>Declare {@code @SessionAttributes({"user", "cart"})} on a controller</li>
 *   <li>After each handler invocation, model attributes matching those names
 *       (or matching the declared types) are automatically stored in the session</li>
 *   <li>On subsequent requests, those attributes are retrieved from the session
 *       and placed back into the model before handler invocation</li>
 *   <li>When the conversation is done, call {@code SessionStatus.setComplete()}
 *       to remove the attributes from the session</li>
 * </ol>
 *
 * <h3>Important distinction:</h3>
 * These are <em>conversational</em> session attributes — temporary, managed by
 * the framework, and cleared on {@code SessionStatus.setComplete()}. For
 * permanent session attributes (like an auth token), use {@code HttpSession}
 * directly or {@code @SessionAttribute} on a method parameter.
 *
 * <h3>Example:</h3>
 * <pre>
 * {@literal @}Controller
 * {@literal @}SessionAttributes("userForm")
 * public class UserWizardController {
 *     // "userForm" model attribute is stored in session automatically
 * }
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No @AliasFor between value() and names() — just value()</li>
 *   <li>No @Inherited behavior across controller hierarchies</li>
 * </ul>
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface SessionAttributes {

    /**
     * Names of model attributes to store in the session.
     *
     * Maps to: {@code SessionAttributes.value()} / {@code names()} (aliased)
     */
    String[] value() default {};

    /**
     * Types of model attributes to store in the session.
     * Any model attribute whose type matches one of these classes
     * will be stored in the session, regardless of its name.
     *
     * Maps to: {@code SessionAttributes.types()}
     */
    Class<?>[] types() default {};
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/SessionAttribute.java` [NEW]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Parameter-level annotation that binds a method parameter to an existing
 * session attribute — for accessing permanent session state like user
 * authentication objects.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.SessionAttribute}
 *
 * <h3>Important distinction from @SessionAttributes:</h3>
 * <ul>
 *   <li>{@code @SessionAttributes} (type-level) = conversational, managed by framework</li>
 *   <li>{@code @SessionAttribute} (parameter-level) = permanent, read-only access</li>
 * </ul>
 *
 * <h3>Example:</h3>
 * <pre>
 * {@literal @}GetMapping("/profile")
 * public String profile({@literal @}SessionAttribute("currentUser") User user) {
 *     // user was put in session by an auth filter — not managed by @SessionAttributes
 * }
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No @AliasFor between value() and name()</li>
 *   <li>No Optional/nullable type detection</li>
 * </ul>
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface SessionAttribute {

    /**
     * The name of the session attribute to bind.
     * Defaults to the parameter name if empty.
     */
    String value() default "";

    /**
     * Whether the session attribute is required.
     * If true and the attribute is missing, an exception is thrown.
     * If false and the attribute is missing, null is injected.
     */
    boolean required() default true;
}
```

#### File: `src/main/java/com/simplespringmvc/session/SessionStatus.java` [NEW]

```java
package com.simplespringmvc.session;

/**
 * Allows a handler method to signal that its session conversation is complete,
 * triggering cleanup of session attributes declared via {@code @SessionAttributes}.
 *
 * Maps to: {@code org.springframework.web.bind.support.SessionStatus}
 *
 * <h3>Usage:</h3>
 * <pre>
 * {@literal @}PostMapping("/finish")
 * public String finish(SessionStatus status) {
 *     status.setComplete();  // clears @SessionAttributes from session
 *     return "redirect:/done";
 * }
 * </pre>
 *
 * The framework creates a SessionStatus per request and checks
 * {@code isComplete()} after handler invocation to decide whether
 * to clean up session attributes.
 */
public interface SessionStatus {

    /**
     * Mark the current handler's session processing as complete,
     * triggering cleanup of session attributes.
     */
    void setComplete();

    /**
     * Whether {@link #setComplete()} has been called.
     */
    boolean isComplete();
}
```

#### File: `src/main/java/com/simplespringmvc/session/SimpleSessionStatus.java` [NEW]

```java
package com.simplespringmvc.session;

/**
 * Default implementation of {@link SessionStatus} backed by a boolean flag.
 *
 * Maps to: {@code org.springframework.web.bind.support.SimpleSessionStatus}
 *
 * Created per request by the HandlerAdapter. After handler execution,
 * the adapter checks {@code isComplete()} — if true, session attributes
 * declared via {@code @SessionAttributes} are removed from the session.
 */
public class SimpleSessionStatus implements SessionStatus {

    private boolean complete = false;

    @Override
    public void setComplete() {
        this.complete = true;
    }

    @Override
    public boolean isComplete() {
        return complete;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/session/SessionAttributeStore.java` [NEW]

```java
package com.simplespringmvc.session;

import jakarta.servlet.http.HttpServletRequest;

/**
 * Strategy interface for storing and retrieving session attributes.
 * Decouples the session attribute management logic from the actual
 * storage mechanism (HttpSession, external store, etc.).
 *
 * Maps to: {@code org.springframework.web.bind.support.SessionAttributeStore}
 *
 * The real interface uses {@code WebRequest} instead of {@code HttpServletRequest}.
 * We use HttpServletRequest directly to avoid the abstraction layer.
 */
public interface SessionAttributeStore {

    /**
     * Store the attribute in the backend session.
     *
     * @param request       the current HTTP request (provides access to HttpSession)
     * @param attributeName the name to store under
     * @param attributeValue the value to store
     */
    void storeAttribute(HttpServletRequest request, String attributeName, Object attributeValue);

    /**
     * Retrieve the attribute from the backend session.
     *
     * @param request       the current HTTP request
     * @param attributeName the name to look up
     * @return the attribute value, or null if not found
     */
    Object retrieveAttribute(HttpServletRequest request, String attributeName);

    /**
     * Remove the attribute from the backend session.
     *
     * @param request       the current HTTP request
     * @param attributeName the name to remove
     */
    void cleanupAttribute(HttpServletRequest request, String attributeName);
}
```

#### File: `src/main/java/com/simplespringmvc/session/DefaultSessionAttributeStore.java` [NEW]

```java
package com.simplespringmvc.session;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;

/**
 * Default implementation of {@link SessionAttributeStore} that delegates
 * to the standard {@code HttpSession}.
 *
 * Maps to: {@code org.springframework.web.bind.support.DefaultSessionAttributeStore}
 *
 * The real version supports a configurable attribute name prefix
 * and uses {@code WebRequest.SCOPE_SESSION} for abstraction.
 * We go straight to {@code HttpSession}.
 *
 * Simplifications:
 * <ul>
 *   <li>No attribute name prefix support</li>
 *   <li>Direct HttpSession access instead of WebRequest</li>
 * </ul>
 */
public class DefaultSessionAttributeStore implements SessionAttributeStore {

    @Override
    public void storeAttribute(HttpServletRequest request, String attributeName, Object attributeValue) {
        request.getSession().setAttribute(attributeName, attributeValue);
    }

    @Override
    public Object retrieveAttribute(HttpServletRequest request, String attributeName) {
        HttpSession session = request.getSession(false);
        if (session == null) {
            return null;
        }
        return session.getAttribute(attributeName);
    }

    @Override
    public void cleanupAttribute(HttpServletRequest request, String attributeName) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            session.removeAttribute(attributeName);
        }
    }
}
```

#### File: `src/main/java/com/simplespringmvc/session/SessionAttributesHandler.java` [NEW]

```java
package com.simplespringmvc.session;

import com.simplespringmvc.annotation.SessionAttributes;

import java.util.Arrays;
import java.util.Collections;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.servlet.http.HttpServletRequest;

/**
 * Manages session attributes for a single controller class based on its
 * {@code @SessionAttributes} declaration. Created once per controller type
 * and cached by the HandlerAdapter.
 *
 * Maps to: {@code org.springframework.web.method.annotation.SessionAttributesHandler}
 *
 * <h3>Responsibilities:</h3>
 * <ul>
 *   <li>Extract {@code @SessionAttributes} metadata (names and types) from the controller class</li>
 *   <li>Determine whether a given model attribute qualifies for session storage</li>
 *   <li>Retrieve all known session attributes before handler invocation</li>
 *   <li>Store matching model attributes after handler invocation</li>
 *   <li>Clean up all managed attributes when {@code SessionStatus.setComplete()} is called</li>
 * </ul>
 *
 * <h3>Name vs Type matching:</h3>
 * Attributes are matched by name (from {@code @SessionAttributes.value()}) OR by type
 * (from {@code @SessionAttributes.types()}). Type-matched attributes are discovered
 * dynamically — when a model attribute's type matches, its name is added to the
 * {@code knownAttributeNames} set for future retrieval.
 *
 * Simplifications:
 * <ul>
 *   <li>No distributed session support (SESSION_KNOWN_ATTRIBUTE tracking)</li>
 *   <li>Uses HttpServletRequest instead of WebRequest</li>
 * </ul>
 */
public class SessionAttributesHandler {

    private final Set<String> attributeNames = new HashSet<>();
    private final Set<Class<?>> attributeTypes = new HashSet<>();
    private final Set<String> knownAttributeNames = ConcurrentHashMap.newKeySet();
    private final SessionAttributeStore sessionAttributeStore;

    /**
     * Create a handler for the given controller type.
     *
     * Maps to: {@code SessionAttributesHandler(Class, SessionAttributeStore)} constructor
     *
     * Extracts @SessionAttributes metadata at construction time. If the annotation
     * is absent, hasSessionAttributes() returns false and all operations are no-ops.
     *
     * @param handlerType the controller class to inspect
     * @param store       the strategy for session attribute storage
     */
    public SessionAttributesHandler(Class<?> handlerType, SessionAttributeStore store) {
        SessionAttributes ann = handlerType.getAnnotation(SessionAttributes.class);
        if (ann != null) {
            Collections.addAll(this.attributeNames, ann.value());
            Collections.addAll(this.attributeTypes, ann.types());
        }
        this.knownAttributeNames.addAll(this.attributeNames);
        this.sessionAttributeStore = store;
    }

    /**
     * Whether the controller declared any session attributes.
     *
     * Maps to: {@code SessionAttributesHandler.hasSessionAttributes()}
     */
    public boolean hasSessionAttributes() {
        return !this.attributeNames.isEmpty() || !this.attributeTypes.isEmpty();
    }

    /**
     * Whether the given attribute name and type match the @SessionAttributes declaration.
     *
     * Maps to: {@code SessionAttributesHandler.isHandlerSessionAttribute()} (line 110)
     *
     * If matched by type, the name is added to knownAttributeNames for future retrieval.
     *
     * @param attributeName the model attribute name
     * @param attributeType the model attribute's runtime type
     * @return true if this attribute should be stored in the session
     */
    public boolean isHandlerSessionAttribute(String attributeName, Class<?> attributeType) {
        if (this.attributeTypes.contains(attributeType)) {
            this.knownAttributeNames.add(attributeName);
            return true;
        }
        return this.attributeNames.contains(attributeName);
    }

    /**
     * Retrieve all known session attributes from the session.
     *
     * Maps to: {@code SessionAttributesHandler.retrieveAttributes()} (line 149)
     *
     * Iterates all known attribute names and retrieves their values from the session.
     * Returns only non-null values.
     *
     * @param request the current HTTP request
     * @return map of attribute name → value for all known attributes in session
     */
    public Map<String, Object> retrieveAttributes(HttpServletRequest request) {
        Map<String, Object> attributes = new HashMap<>();
        for (String name : this.knownAttributeNames) {
            Object value = this.sessionAttributeStore.retrieveAttribute(request, name);
            if (value != null) {
                attributes.put(name, value);
            }
        }
        return attributes;
    }

    /**
     * Store model attributes that match the @SessionAttributes declaration into the session.
     *
     * Maps to: {@code SessionAttributesHandler.storeAttributes()} (line 127)
     *
     * Iterates the given attributes map and stores only those that match
     * by name or type.
     *
     * @param request    the current HTTP request
     * @param attributes the model attributes to potentially store
     */
    public void storeAttributes(HttpServletRequest request, Map<String, ?> attributes) {
        attributes.forEach((name, value) -> {
            if (value != null && isHandlerSessionAttribute(name, value.getClass())) {
                this.sessionAttributeStore.storeAttribute(request, name, value);
            }
        });
    }

    /**
     * Remove all known session attributes from the session.
     * Called when {@code SessionStatus.setComplete()} has been invoked.
     *
     * Maps to: {@code SessionAttributesHandler.cleanupAttributes()} (line 174)
     *
     * @param request the current HTTP request
     */
    public void cleanupAttributes(HttpServletRequest request) {
        for (String name : this.knownAttributeNames) {
            this.sessionAttributeStore.cleanupAttribute(request, name);
        }
    }

    /**
     * Returns the declared attribute names (for testing/inspection).
     */
    public Set<String> getAttributeNames() {
        return Set.copyOf(attributeNames);
    }

    /**
     * Returns the declared attribute types (for testing/inspection).
     */
    public Set<Class<?>> getAttributeTypes() {
        return Set.copyOf(attributeTypes);
    }

    /**
     * Returns all known attribute names — including those discovered via type matching
     * (for testing/inspection).
     */
    public Set<String> getKnownAttributeNames() {
        return Set.copyOf(knownAttributeNames);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/flash/FlashMap.java` [NEW]

```java
package com.simplespringmvc.flash;

import java.util.LinkedHashMap;

/**
 * A HashMap that carries flash attributes along with a target request path
 * and an expiration time. Flash attributes survive exactly one redirect —
 * they're saved before the redirect and consumed on the target request.
 *
 * Maps to: {@code org.springframework.web.servlet.FlashMap}
 *
 * <h3>The Post/Redirect/Get pattern:</h3>
 * <pre>
 *   POST /order/submit
 *     → save flash attribute "message" = "Order placed!"
 *     → redirect to GET /order/confirmation
 *   GET /order/confirmation
 *     → flash map matches this request path
 *     → "message" is available in the model
 *     → flash map is consumed (removed from storage)
 * </pre>
 *
 * <h3>Target path matching:</h3>
 * If {@code targetRequestPath} is set, the FlashMap only matches requests
 * to that exact path. If null, it matches any request (less precise but
 * simpler — the first request after the redirect consumes it).
 *
 * <h3>Expiration:</h3>
 * Flash maps expire after a configurable timeout (default 180 seconds).
 * This prevents orphaned flash maps from accumulating in the session
 * if the redirect target is never reached.
 *
 * Simplifications:
 * <ul>
 *   <li>No target request parameters matching (only path matching)</li>
 *   <li>No Comparable implementation for priority sorting</li>
 * </ul>
 */
public class FlashMap extends LinkedHashMap<String, Object> {

    private String targetRequestPath;
    private long expirationTime;

    /**
     * Set the target request path that this FlashMap should match.
     *
     * Maps to: {@code FlashMap.setTargetRequestPath()} which normalizes
     * the path and strips trailing slashes.
     *
     * @param path the expected request path (e.g., "/order/confirmation")
     */
    public void setTargetRequestPath(String path) {
        this.targetRequestPath = path;
    }

    /**
     * Return the target request path, or null if not set.
     */
    public String getTargetRequestPath() {
        return this.targetRequestPath;
    }

    /**
     * Start the expiration countdown.
     *
     * Maps to: {@code FlashMap.startExpirationPeriod(int)} (line 99)
     *
     * @param timeToLive the number of seconds before this FlashMap expires
     */
    public void startExpirationPeriod(int timeToLive) {
        this.expirationTime = System.currentTimeMillis() + timeToLive * 1000L;
    }

    /**
     * Whether this FlashMap has expired.
     *
     * Maps to: {@code FlashMap.isExpired()} (line 121)
     */
    public boolean isExpired() {
        return this.expirationTime > 0 && System.currentTimeMillis() > this.expirationTime;
    }

    /**
     * Return the expiration time (milliseconds since epoch), or 0 if not set.
     */
    public long getExpirationTime() {
        return this.expirationTime;
    }

    @Override
    public String toString() {
        return "FlashMap [attributes=" + super.toString()
                + ", targetRequestPath=" + targetRequestPath + "]";
    }
}
```

#### File: `src/main/java/com/simplespringmvc/flash/FlashMapManager.java` [NEW]

```java
package com.simplespringmvc.flash;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * Strategy interface for managing {@link FlashMap} instances — retrieving
 * flash attributes from a previous request and saving them for the next.
 *
 * Maps to: {@code org.springframework.web.servlet.FlashMapManager}
 *
 * Called by the DispatcherServlet at the beginning and end of each request:
 * <ul>
 *   <li>{@code retrieveAndUpdate()} at request start — finds and consumes matching flash maps</li>
 *   <li>{@code saveOutputFlashMap()} before redirect — persists flash attributes for the next request</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No AbstractFlashMapManager template method pattern — direct implementation</li>
 * </ul>
 */
public interface FlashMapManager {

    /**
     * Find the FlashMap that matches the current request, remove it from
     * storage, and clean up any expired flash maps.
     *
     * Maps to: {@code FlashMapManager.retrieveAndUpdate()}
     *
     * @param request  the current HTTP request
     * @param response the current HTTP response
     * @return the matching FlashMap, or null if none match
     */
    FlashMap retrieveAndUpdate(HttpServletRequest request, HttpServletResponse response);

    /**
     * Save the output FlashMap (with flash attributes and target path)
     * to persistent storage so the next request can retrieve it.
     *
     * Maps to: {@code FlashMapManager.saveOutputFlashMap()}
     *
     * @param flashMap the FlashMap to save
     * @param request  the current HTTP request
     * @param response the current HTTP response
     */
    void saveOutputFlashMap(FlashMap flashMap, HttpServletRequest request, HttpServletResponse response);
}
```

#### File: `src/main/java/com/simplespringmvc/flash/SessionFlashMapManager.java` [NEW]

```java
package com.simplespringmvc.flash;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

/**
 * {@link FlashMapManager} implementation that stores flash maps in the
 * HTTP session.
 *
 * Maps to: {@code org.springframework.web.servlet.support.SessionFlashMapManager}
 * which extends {@code AbstractFlashMapManager}.
 *
 * <h3>Storage mechanism:</h3>
 * All flash maps are stored as a single {@code List<FlashMap>} in the session
 * under a well-known key. This allows multiple flash maps to coexist (e.g.,
 * when the user opens multiple tabs that perform redirects).
 *
 * <h3>Matching algorithm:</h3>
 * A FlashMap matches the current request if:
 * <ol>
 *   <li>It hasn't expired</li>
 *   <li>Its target request path matches the current request URI
 *       (or the FlashMap has no target path — matches any request)</li>
 * </ol>
 *
 * <h3>Cleanup:</h3>
 * On each {@code retrieveAndUpdate()} call, expired flash maps are removed.
 * This prevents orphaned maps from accumulating when redirect targets
 * are never reached.
 *
 * Simplifications:
 * <ul>
 *   <li>No AbstractFlashMapManager base class — single concrete class</li>
 *   <li>No target request parameter matching (only path matching)</li>
 *   <li>No session mutex synchronization</li>
 *   <li>No URL path normalization</li>
 * </ul>
 */
public class SessionFlashMapManager implements FlashMapManager {

    private static final String FLASH_MAPS_SESSION_ATTRIBUTE =
            SessionFlashMapManager.class.getName() + ".FLASH_MAPS";

    /**
     * Default time-to-live for flash maps, in seconds.
     *
     * Maps to: {@code AbstractFlashMapManager.flashMapTimeout} (default 180)
     */
    private int flashMapTimeout = 180;

    @Override
    public FlashMap retrieveAndUpdate(HttpServletRequest request, HttpServletResponse response) {
        List<FlashMap> allFlashMaps = retrieveFlashMaps(request);
        if (allFlashMaps == null || allFlashMaps.isEmpty()) {
            return null;
        }

        FlashMap match = null;
        boolean mapsModified = false;

        // Iterate and find matching + expired flash maps
        Iterator<FlashMap> iterator = allFlashMaps.iterator();
        while (iterator.hasNext()) {
            FlashMap flashMap = iterator.next();
            if (flashMap.isExpired()) {
                iterator.remove();
                mapsModified = true;
            } else if (match == null && isFlashMapForRequest(flashMap, request)) {
                match = flashMap;
                iterator.remove();
                mapsModified = true;
            }
        }

        // Persist changes if any flash maps were removed
        if (mapsModified) {
            updateFlashMaps(allFlashMaps, request);
        }

        return match;
    }

    @Override
    public void saveOutputFlashMap(FlashMap flashMap, HttpServletRequest request,
                                    HttpServletResponse response) {
        if (flashMap.isEmpty()) {
            return;
        }

        flashMap.startExpirationPeriod(this.flashMapTimeout);

        List<FlashMap> allFlashMaps = retrieveFlashMaps(request);
        if (allFlashMaps == null) {
            allFlashMaps = new ArrayList<>();
        }
        allFlashMaps.add(flashMap);
        updateFlashMaps(allFlashMaps, request);
    }

    /**
     * Check if a FlashMap's target path matches the current request.
     *
     * Maps to: {@code AbstractFlashMapManager.isFlashMapForRequest()} (line 146)
     *
     * If the FlashMap has no target path, it matches any request.
     * Otherwise, the request URI must match exactly.
     */
    private boolean isFlashMapForRequest(FlashMap flashMap, HttpServletRequest request) {
        String targetPath = flashMap.getTargetRequestPath();
        if (targetPath != null) {
            String requestUri = request.getRequestURI();
            if (!requestUri.equals(targetPath) && !requestUri.equals(targetPath + "/")) {
                return false;
            }
        }
        return true;
    }

    /**
     * Retrieve the list of flash maps from the session.
     *
     * Maps to: {@code SessionFlashMapManager.retrieveFlashMaps()}
     */
    @SuppressWarnings("unchecked")
    private List<FlashMap> retrieveFlashMaps(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        if (session == null) {
            return null;
        }
        return (List<FlashMap>) session.getAttribute(FLASH_MAPS_SESSION_ATTRIBUTE);
    }

    /**
     * Store the list of flash maps in the session.
     *
     * Maps to: {@code SessionFlashMapManager.updateFlashMaps()}
     */
    private void updateFlashMaps(List<FlashMap> flashMaps, HttpServletRequest request) {
        HttpSession session = request.getSession();
        if (flashMaps.isEmpty()) {
            session.removeAttribute(FLASH_MAPS_SESSION_ATTRIBUTE);
        } else {
            session.setAttribute(FLASH_MAPS_SESSION_ATTRIBUTE, flashMaps);
        }
    }

    /**
     * Set the flash map timeout in seconds.
     */
    public void setFlashMapTimeout(int flashMapTimeout) {
        this.flashMapTimeout = flashMapTimeout;
    }

    /**
     * Return the configured timeout (for testing).
     */
    public int getFlashMapTimeout() {
        return this.flashMapTimeout;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/flash/SimpleRedirectAttributes.java` [NEW]

```java
package com.simplespringmvc.flash;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Holds flash attributes that should survive a redirect — the bridge
 * between handler method code and the FlashMap infrastructure.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.support.RedirectAttributesModelMap}
 * which implements {@code RedirectAttributes} (extends {@code Model}).
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>Controller declares a {@code SimpleRedirectAttributes} parameter</li>
 *   <li>Controller calls {@code addFlashAttribute("msg", "Success!")}</li>
 *   <li>Controller returns {@code "redirect:/target"}</li>
 *   <li>DispatcherServlet copies flash attributes into the output FlashMap</li>
 *   <li>FlashMapManager saves the FlashMap in the session</li>
 *   <li>On the redirected request, the FlashMap is retrieved and its
 *       attributes are merged into the model</li>
 * </ol>
 *
 * <h3>Flash vs regular attributes:</h3>
 * In the real framework, {@code RedirectAttributes} also supports regular
 * attributes that become URL query parameters on the redirect. We only
 * support flash attributes (session-backed, one-time-use).
 *
 * Simplifications:
 * <ul>
 *   <li>Does not extend Model — standalone class</li>
 *   <li>No regular redirect attributes (URL query params)</li>
 *   <li>No DataBinder-based value formatting</li>
 *   <li>No convention-based attribute naming</li>
 * </ul>
 */
public class SimpleRedirectAttributes {

    private final Map<String, Object> flashAttributes = new LinkedHashMap<>();

    /**
     * Add a flash attribute that will survive the redirect.
     *
     * Maps to: {@code RedirectAttributes.addFlashAttribute(String, Object)}
     *
     * @param name  the attribute name
     * @param value the attribute value
     * @return this instance for fluent chaining
     */
    public SimpleRedirectAttributes addFlashAttribute(String name, Object value) {
        this.flashAttributes.put(name, value);
        return this;
    }

    /**
     * Return all flash attributes.
     *
     * Maps to: {@code RedirectAttributes.getFlashAttributes()}
     */
    public Map<String, Object> getFlashAttributes() {
        return this.flashAttributes;
    }

    @Override
    public String toString() {
        return "SimpleRedirectAttributes [flash=" + flashAttributes + "]";
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SessionAttributeArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.SessionAttribute;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;

/**
 * Resolves method parameters annotated with {@link SessionAttribute} by
 * looking up the named attribute in the HTTP session.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.SessionAttributeMethodArgumentResolver}
 *
 * <h3>Important distinction:</h3>
 * This resolves permanent session attributes (e.g., auth tokens set by a filter)
 * — NOT the conversational attributes managed by {@code @SessionAttributes}.
 *
 * <h3>Example:</h3>
 * <pre>
 * {@literal @}GetMapping("/profile")
 * public String profile({@literal @}SessionAttribute("currentUser") User user) {
 *     // user was stored in session by an auth filter
 * }
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>No AbstractNamedValueMethodArgumentResolver base class</li>
 *   <li>No type conversion of session attribute values</li>
 *   <li>No Optional parameter support</li>
 * </ul>
 */
public class SessionAttributeArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(SessionAttribute.class);
    }

    /**
     * Retrieve the session attribute by name.
     *
     * Maps to: {@code SessionAttributeMethodArgumentResolver.resolveName()}
     *
     * The attribute name comes from {@code @SessionAttribute.value()}, or
     * falls back to the parameter name if not specified.
     *
     * @throws IllegalStateException if the attribute is required but missing
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        SessionAttribute annotation = parameter.getParameterAnnotation(SessionAttribute.class);

        String attributeName = annotation.value();
        if (attributeName.isEmpty()) {
            attributeName = parameter.getParameterName();
        }

        HttpSession session = request.getSession(false);
        Object value = (session != null) ? session.getAttribute(attributeName) : null;

        if (value == null && annotation.required()) {
            throw new IllegalStateException(
                    "Required session attribute '" + attributeName + "' of type "
                            + parameter.getParameterType().getName() + " is missing. "
                            + "Ensure it was set in the session before this request.");
        }

        return value;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SessionStatusArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.session.SessionStatus;
import jakarta.servlet.http.HttpServletRequest;

/**
 * Resolves method parameters of type {@link SessionStatus} by returning
 * the per-request SessionStatus instance stored in a request attribute.
 *
 * Maps to: {@code org.springframework.web.method.annotation.SessionStatusMethodArgumentResolver}
 *
 * <h3>How it works:</h3>
 * The {@code SimpleHandlerAdapter} creates a {@code SimpleSessionStatus} for each
 * request and stores it as a request attribute. This resolver retrieves it.
 * After handler invocation, the adapter checks {@code sessionStatus.isComplete()}
 * to decide whether to clean up session attributes.
 *
 * <h3>Example:</h3>
 * <pre>
 * {@literal @}PostMapping("/finish")
 * public String finish(SessionStatus status) {
 *     status.setComplete();  // triggers cleanup of @SessionAttributes
 *     return "redirect:/done";
 * }
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Reads from request attribute instead of ModelAndViewContainer</li>
 *   <li>No annotation required — detected purely by parameter type</li>
 * </ul>
 */
public class SessionStatusArgumentResolver implements HandlerMethodArgumentResolver {

    /**
     * Request attribute key where the SimpleHandlerAdapter stores the
     * per-request SessionStatus instance.
     */
    public static final String SESSION_STATUS_ATTRIBUTE =
            SessionStatusArgumentResolver.class.getName() + ".SESSION_STATUS";

    /**
     * Supports parameters whose type is exactly {@link SessionStatus}.
     *
     * Maps to: {@code SessionStatusMethodArgumentResolver.supportsParameter()}
     * Real version: {@code SessionStatus.class == parameter.getParameterType()}
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return SessionStatus.class.isAssignableFrom(parameter.getParameterType());
    }

    /**
     * Return the SessionStatus instance from the request attribute.
     *
     * Maps to: {@code SessionStatusMethodArgumentResolver.resolveArgument()}
     * Real version: returns {@code mavContainer.getSessionStatus()}
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        Object sessionStatus = request.getAttribute(SESSION_STATUS_ATTRIBUTE);
        if (sessionStatus == null) {
            throw new IllegalStateException(
                    "No SessionStatus found in request. This resolver requires "
                            + "SimpleHandlerAdapter to set up the SessionStatus before "
                            + "argument resolution.");
        }
        return sessionStatus;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/RedirectAttributesArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.flash.SimpleRedirectAttributes;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;

/**
 * Resolves method parameters of type {@link SimpleRedirectAttributes} for
 * adding flash attributes that survive a redirect.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RedirectAttributesMethodArgumentResolver}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>Creates a new {@code SimpleRedirectAttributes} instance</li>
 *   <li>Stores it as a request attribute so the DispatcherServlet can
 *       retrieve it during redirect processing</li>
 *   <li>Returns it to the handler method for adding flash attributes</li>
 * </ol>
 *
 * <h3>Example:</h3>
 * <pre>
 * {@literal @}PostMapping("/save")
 * public String save(SimpleRedirectAttributes redirectAttrs) {
 *     redirectAttrs.addFlashAttribute("message", "Saved!");
 *     return "redirect:/list";
 * }
 * </pre>
 *
 * Simplifications:
 * <ul>
 *   <li>Creates SimpleRedirectAttributes instead of RedirectAttributesModelMap</li>
 *   <li>No DataBinder/WebDataBinderFactory for value formatting</li>
 *   <li>No ModelAndViewContainer.setRedirectModel()</li>
 * </ul>
 */
public class RedirectAttributesArgumentResolver implements HandlerMethodArgumentResolver {

    /**
     * Request attribute key where the redirect attributes instance is stored
     * for later retrieval by the DispatcherServlet during redirect processing.
     */
    public static final String REDIRECT_ATTRIBUTES_ATTRIBUTE =
            RedirectAttributesArgumentResolver.class.getName() + ".REDIRECT_ATTRIBUTES";

    /**
     * Supports parameters whose type is {@link SimpleRedirectAttributes}.
     *
     * Maps to: {@code RedirectAttributesMethodArgumentResolver.supportsParameter()}
     * Real version checks for {@code RedirectAttributes} interface.
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return SimpleRedirectAttributes.class.isAssignableFrom(parameter.getParameterType());
    }

    /**
     * Create a new SimpleRedirectAttributes, store it as a request attribute,
     * and return it.
     *
     * Maps to: {@code RedirectAttributesMethodArgumentResolver.resolveArgument()}
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        SimpleRedirectAttributes redirectAttributes = new SimpleRedirectAttributes();
        request.setAttribute(REDIRECT_ATTRIBUTES_ATTRIBUTE, redirectAttributes);
        return redirectAttributes;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/annotation/ModelAttribute.java` [MODIFIED]

```java
package com.simplespringmvc.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Annotation that binds a method parameter to a named model attribute,
 * populated via data binding from request parameters.
 *
 * Maps to: {@code org.springframework.web.bind.annotation.ModelAttribute}
 *
 * When placed on a handler method parameter, the framework will:
 * <ol>
 *   <li>Create an instance of the parameter type using its default constructor</li>
 *   <li>Iterate all request parameters and bind matching ones to the object's properties</li>
 *   <li>Convert String parameter values to the property's declared type via ConversionService</li>
 *   <li>Pass the populated object as the method argument</li>
 * </ol>
 *
 * <h3>Example:</h3>
 * <pre>
 * {@literal @}PostMapping("/users")
 * public String createUser({@literal @}ModelAttribute UserForm form) {
 *     // form.getName(), form.getAge() etc. are populated from request params
 * }
 * </pre>
 *
 * The real annotation also supports:
 * <ul>
 *   <li>Method-level placement (to add attributes to the model before every handler invocation)</li>
 *   <li>{@code name} / {@code value} attributes for explicit model attribute naming</li>
 *   <li>{@code binding} attribute to disable data binding</li>
 * </ul>
 *
 * <h3>ch18 Enhancement:</h3>
 * Added {@code value()} attribute so the attribute name can be specified explicitly.
 * This is needed for {@code @SessionAttributes} integration — the session attribute
 * name must match the {@code @ModelAttribute} name for the framework to retrieve
 * existing instances from the session instead of creating new ones.
 *
 * Simplifications:
 * <ul>
 *   <li>Parameter-level only — no method-level model population</li>
 *   <li>No {@code binding} flag — binding is always enabled</li>
 * </ul>
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ModelAttribute {

    /**
     * The name of the model attribute to bind.
     * If empty, the name is derived from the parameter type
     * (e.g., {@code UserForm} → "userForm").
     *
     * <h3>ch18 Enhancement:</h3>
     * Added to support explicit naming for session attribute matching.
     *
     * Maps to: {@code ModelAttribute.value()} / {@code name()} (aliased in real framework)
     */
    String value() default "";
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/ModelAttributeArgumentResolver.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.ModelAttribute;
import com.simplespringmvc.binding.SimpleDataBinder;
import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.validation.BindingResult;
import com.simplespringmvc.validation.MethodArgumentNotValidException;
import com.simplespringmvc.validation.SimpleValidatorAdapter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.Valid;

import java.util.Map;

/**
 * Resolves method arguments annotated with {@link ModelAttribute} by creating
 * a new instance of the parameter type, binding request parameters to it,
 * and optionally validating if {@code @Valid} is present.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.ServletModelAttributeMethodProcessor}
 * which extends {@code org.springframework.web.method.annotation.ModelAttributeMethodProcessor}
 *
 * <h3>How it works:</h3>
 * <ol>
 *   <li>{@code supportsParameter()} checks if the parameter has {@code @ModelAttribute}</li>
 *   <li>{@code resolveArgument()} delegates to {@link SimpleDataBinder} to create
 *       and populate the target POJO from request parameters</li>
 *   <li>If the parameter has {@code @Valid}, runs Jakarta Bean Validation</li>
 *   <li>If validation errors exist, throws {@link MethodArgumentNotValidException}</li>
 * </ol>
 *
 * <h3>ch18 Enhancement:</h3>
 * <ul>
 *   <li>Added session attribute lookup before creating a new instance — if an attribute
 *       with the same name exists in the session (via {@code @SessionAttributes}), it is
 *       retrieved and request parameters are bound to the existing instance</li>
 *   <li>Added {@code getAttributeName()} that reads explicit name from
 *       {@code @ModelAttribute.value()} or derives from the parameter type</li>
 * </ul>
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   ModelAttributeMethodProcessor.resolveArgument():
 *     1. Determine attribute name from @ModelAttribute or parameter type
 *     2. Check if attribute exists in the model (ModelAndViewContainer)
 *     3. If not, create via createAttribute() or DataBinder.construct()
 *     4. Create WebDataBinder for the attribute
 *     5. Call bindRequestParameters(binder, request)
 *        → ServletRequestDataBinder.bind(servletRequest)
 *        → Extracts params into MutablePropertyValues
 *        → BeanWrapperImpl sets properties via setters
 *     6. validateIfApplicable(binder, parameter)
 *        → Iterates parameter annotations
 *        → Uses ValidationAnnotationUtils.determineValidationHints()
 *        → Calls binder.validate(hints) → DataBinder.validate()
 *        → Delegates to SpringValidatorAdapter → jakarta.validation.Validator
 *     7. Check bindingResult.hasErrors()
 *        → If next param is Errors/BindingResult → pass through
 *        → Otherwise → throw MethodArgumentNotValidException
 *     8. Add attribute + BindingResult to model
 * </pre>
 *
 * <h3>The "annotationNotRequired" mode:</h3>
 * The real framework creates TWO instances of ModelAttributeMethodProcessor:
 * <ul>
 *   <li>One with {@code annotationNotRequired=false} — only handles @ModelAttribute</li>
 *   <li>One with {@code annotationNotRequired=true} — handles any non-simple type as a
 *       catch-all at the END of the resolver chain</li>
 * </ul>
 * This means in the real framework, a method parameter like {@code UserForm form}
 * (without @ModelAttribute) is STILL bound from request parameters. We require
 * the explicit annotation for clarity.
 *
 * Simplifications:
 * <ul>
 *   <li>Requires explicit {@code @ModelAttribute} — no "default resolution" mode</li>
 *   <li>No BindingResult parameter support — always throws on validation failure</li>
 *   <li>No validation groups ({@code @Validated(Group.class)}) — only {@code @Valid}</li>
 * </ul>
 */
public class ModelAttributeArgumentResolver implements HandlerMethodArgumentResolver {

    /**
     * Request attribute key where the SimpleHandlerAdapter stores session attributes
     * retrieved before argument resolution. The resolver checks this map for
     * existing instances before creating new ones.
     *
     * <h3>ch18 Enhancement:</h3>
     * Added to support session attribute lookup.
     */
    public static final String SESSION_ATTRIBUTES_KEY =
            ModelAttributeArgumentResolver.class.getName() + ".SESSION_ATTRIBUTES";

    private final SimpleDataBinder dataBinder;
    private final SimpleValidatorAdapter validatorAdapter;

    public ModelAttributeArgumentResolver(ConversionService conversionService) {
        this.dataBinder = new SimpleDataBinder(conversionService);
        this.validatorAdapter = new SimpleValidatorAdapter();
    }

    /**
     * Supports parameters annotated with {@code @ModelAttribute}.
     *
     * Maps to: {@code ModelAttributeMethodProcessor.supportsParameter()} (line 91)
     * The real version: {@code parameter.hasParameterAnnotation(ModelAttribute.class)
     *   || (annotationNotRequired && !BeanUtils.isSimpleProperty(parameterType))}
     *
     * We only support the explicit annotation case.
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(ModelAttribute.class);
    }

    /**
     * Create an instance of the parameter type (or retrieve from session), bind
     * request parameters to it, and validate if {@code @Valid} is present.
     *
     * Maps to: {@code ModelAttributeMethodProcessor.resolveArgument()} (line 106)
     *
     * <h3>ch18 Enhancement:</h3>
     * Before creating a new instance, checks session attributes (stored as a request
     * attribute by the SimpleHandlerAdapter) for an existing instance with the same
     * name. If found, binds request parameters to the existing instance instead of
     * creating a new one. This is how multi-step form wizards work — the same object
     * is retrieved from the session, updated with new form values, and stored back.
     *
     * The real flow:
     * <pre>
     *   1. name = ModelFactory.getNameForParameter(parameter)
     *   2. if (mavContainer.containsAttribute(name)) → get from model
     *   3. else → createAttribute() or DataBinder.construct()
     *   4. binder = binderFactory.createBinder(request, attribute, name)
     *   5. binder.bind(servletRequest)       // property binding
     *   6. validateIfApplicable(binder, parameter)  // @Valid → jakarta.validation
     *   7. check bindingResult.hasErrors()   → throw MethodArgumentNotValidException
     *   8. add attribute + bindingResult to model
     * </pre>
     *
     * We collapse this to: find-or-create → bind → validate → return (or throw).
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request) throws Exception {
        Class<?> targetType = parameter.getParameterType();
        String attrName = getAttributeName(parameter);

        // ch18: Check session attributes for existing instance
        Object target = findInSessionAttributes(attrName, request);

        if (target != null) {
            // Bind request parameters to existing session instance
            dataBinder.bind(target, request);
        } else {
            // Create new instance and bind
            target = dataBinder.bind(targetType, request);
        }

        // Step 6: validateIfApplicable — check for @Valid on the parameter
        validateIfApplicable(target, parameter);

        return target;
    }

    /**
     * Determine the attribute name from the {@code @ModelAttribute} annotation
     * or derive it from the parameter type.
     *
     * <h3>ch18 Enhancement:</h3>
     * Added to support explicit naming for session attribute matching.
     *
     * Maps to: {@code ModelFactory.getNameForParameter()} which calls
     * {@code Conventions.getVariableNameForParameter()} →
     * uncapitalize(ClassName)
     *
     * @param parameter the method parameter
     * @return the attribute name
     */
    String getAttributeName(MethodParameter parameter) {
        ModelAttribute annotation = parameter.getParameterAnnotation(ModelAttribute.class);
        if (annotation != null && !annotation.value().isEmpty()) {
            return annotation.value();
        }
        return uncapitalize(parameter.getParameterType().getSimpleName());
    }

    /**
     * Look for an existing instance in the session attributes map
     * (stored as a request attribute by the SimpleHandlerAdapter).
     *
     * <h3>ch18 Enhancement:</h3>
     * Added to support session attribute retrieval before creating new instances.
     *
     * @param attributeName the attribute name to look for
     * @param request       the current HTTP request
     * @return the existing instance, or null if not found
     */
    @SuppressWarnings("unchecked")
    private Object findInSessionAttributes(String attributeName, HttpServletRequest request) {
        Map<String, Object> sessionAttributes =
                (Map<String, Object>) request.getAttribute(SESSION_ATTRIBUTES_KEY);
        if (sessionAttributes != null) {
            return sessionAttributes.get(attributeName);
        }
        return null;
    }

    /**
     * If the parameter is annotated with {@code @Valid}, run Jakarta Bean Validation
     * and throw {@link MethodArgumentNotValidException} if violations are found.
     *
     * Maps to: {@code ModelAttributeMethodProcessor.validateIfApplicable()} (line 246)
     *
     * <h3>Real framework flow:</h3>
     * <pre>
     *   for (Annotation ann : parameter.getParameterAnnotations()) {
     *       Object[] validationHints = ValidationAnnotationUtils.determineValidationHints(ann);
     *       if (validationHints != null) {
     *           binder.validate(validationHints);  // → DataBinder.validate()
     *           break;                             // only first match
     *       }
     *   }
     * </pre>
     *
     * {@code ValidationAnnotationUtils.determineValidationHints()} detects:
     * <ol>
     *   <li>{@code @Validated} — returns validation groups from value()</li>
     *   <li>{@code @jakarta.validation.Valid} — returns empty array (no groups)</li>
     *   <li>Any annotation meta-annotated with {@code @Validated}</li>
     *   <li>Any annotation whose simple name starts with "Valid"</li>
     * </ol>
     *
     * We simplify to: just check for {@code @jakarta.validation.Valid}.
     *
     * <h3>The "isBindExceptionRequired" check:</h3>
     * After validation, the real framework checks if the NEXT parameter in the method
     * signature is of type {@code Errors}/{@code BindingResult}. If so, errors are
     * silently passed through (the controller handles them). If not, the exception
     * is thrown. We always throw — no BindingResult parameter support.
     */
    private void validateIfApplicable(Object target, MethodParameter parameter) throws MethodArgumentNotValidException {
        if (!parameter.hasParameterAnnotation(Valid.class)) {
            return;
        }

        // Derive object name from type (e.g., UserForm → "userForm")
        // Maps to: ModelFactory.getNameForParameter() which uses
        // Conventions.getVariableNameForParameter() → uncapitalize(ClassName)
        String objectName = uncapitalize(target.getClass().getSimpleName());

        BindingResult bindingResult = new BindingResult(target, objectName);
        validatorAdapter.validate(target, bindingResult);

        if (bindingResult.hasErrors()) {
            throw new MethodArgumentNotValidException(parameter, bindingResult);
        }
    }

    /**
     * Uncapitalize the first character: "UserForm" → "userForm".
     */
    private static String uncapitalize(String name) {
        if (name == null || name.isEmpty()) {
            return name;
        }
        return Character.toLowerCase(name.charAt(0)) + name.substring(1);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [MODIFIED]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.convert.ConversionService;
import com.simplespringmvc.convert.SimpleConversionService;
import com.simplespringmvc.converter.HttpMessageConverter;
import com.simplespringmvc.converter.JacksonMessageConverter;
import com.simplespringmvc.converter.PlainTextMessageConverter;
import com.simplespringmvc.http.ContentNegotiationManager;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.session.DefaultSessionAttributeStore;
import com.simplespringmvc.session.SessionAttributeStore;
import com.simplespringmvc.session.SessionAttributesHandler;
import com.simplespringmvc.session.SimpleSessionStatus;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.lang.reflect.InvocationTargetException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * The adapter that bridges the DispatcherServlet's generic handler invocation
 * to our specific HandlerMethod-based invocation pipeline.
 *
 * Maps to: {@code org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter}
 *
 * <h3>ch18 Enhancement:</h3>
 * <ul>
 *   <li>Manages the {@code @SessionAttributes} lifecycle around handler invocation:
 *       retrieves session attributes before invocation, stores matching model
 *       attributes after invocation, and cleans up on SessionStatus.setComplete()</li>
 *   <li>Caches {@link SessionAttributesHandler} per controller class for efficiency</li>
 *   <li>Registers three new argument resolvers: {@link SessionAttributeArgumentResolver},
 *       {@link SessionStatusArgumentResolver}, and {@link RedirectAttributesArgumentResolver}</li>
 *   <li>Sets up per-request {@code SimpleSessionStatus} before argument resolution</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Single adapter instance (not a chain of adapters)</li>
 *   <li>No ModelAndViewContainer — return value handlers return ModelAndView directly</li>
 *   <li>No ModelFactory — session attribute lifecycle is managed directly</li>
 *   <li>No WebDataBinderFactory</li>
 * </ul>
 */
public class SimpleHandlerAdapter implements HandlerAdapter {

    private final HandlerMethodArgumentResolverComposite argumentResolvers =
            new HandlerMethodArgumentResolverComposite();
    private final HandlerMethodReturnValueHandlerComposite returnValueHandlers =
            new HandlerMethodReturnValueHandlerComposite();
    private final List<HttpMessageConverter> messageConverters = new ArrayList<>();
    private final ConversionService conversionService;
    private final ContentNegotiationManager contentNegotiationManager;

    /**
     * ch18: Cache of SessionAttributesHandler per controller class.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.sessionAttributesHandlerCache} (line 194)
     * The real version uses a ConcurrentHashMap keyed by controller type.
     */
    private final Map<Class<?>, SessionAttributesHandler> sessionAttributesHandlerCache =
            new ConcurrentHashMap<>(64);

    /**
     * ch18: Strategy for session attribute storage.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.sessionAttributeStore}
     */
    private final SessionAttributeStore sessionAttributeStore = new DefaultSessionAttributeStore();

    public SimpleHandlerAdapter() {
        // ch14: Create the ContentNegotiationManager for Accept header parsing
        contentNegotiationManager = new ContentNegotiationManager();

        // ch14: Register both JSON and plain-text converters.
        // ORDER MATTERS: JacksonMessageConverter first means JSON is the default
        // when Accept is */* (most common case). PlainTextMessageConverter handles
        // text/plain requests for String return values.
        messageConverters.add(new JacksonMessageConverter());
        messageConverters.add(new PlainTextMessageConverter());

        conversionService = new SimpleConversionService();

        // ch18: Session-aware resolvers come FIRST — they match by annotation/type
        // and should take priority over generic resolvers.
        argumentResolvers.addResolver(new SessionAttributeArgumentResolver());
        argumentResolvers.addResolver(new SessionStatusArgumentResolver());
        argumentResolvers.addResolver(new RedirectAttributesArgumentResolver());

        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
        // ch15: ModelAttributeArgumentResolver AFTER specific resolvers — they take priority
        argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));

        // ch14: ResponseBodyReturnValueHandler now takes a ContentNegotiationManager
        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(
                messageConverters, contentNegotiationManager));
        returnValueHandlers.addHandler(new ModelAndViewReturnValueHandler());
        returnValueHandlers.addHandler(new ViewNameReturnValueHandler());
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    public SimpleHandlerAdapter(List<HandlerMethodArgumentResolver> customResolvers) {
        contentNegotiationManager = new ContentNegotiationManager();

        messageConverters.add(new JacksonMessageConverter());
        messageConverters.add(new PlainTextMessageConverter());

        conversionService = new SimpleConversionService();

        if (customResolvers != null) {
            argumentResolvers.addResolvers(customResolvers);
        }
        // ch18: Session-aware resolvers
        argumentResolvers.addResolver(new SessionAttributeArgumentResolver());
        argumentResolvers.addResolver(new SessionStatusArgumentResolver());
        argumentResolvers.addResolver(new RedirectAttributesArgumentResolver());

        argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
        argumentResolvers.addResolver(new RequestBodyArgumentResolver(messageConverters));
        // ch15: ModelAttributeArgumentResolver AFTER specific resolvers
        argumentResolvers.addResolver(new ModelAttributeArgumentResolver(conversionService));

        returnValueHandlers.addHandler(new ResponseBodyReturnValueHandler(
                messageConverters, contentNegotiationManager));
        returnValueHandlers.addHandler(new ModelAndViewReturnValueHandler());
        returnValueHandlers.addHandler(new ViewNameReturnValueHandler());
        returnValueHandlers.addHandler(new StringReturnValueHandler());
    }

    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerMethod;
    }

    /**
     * Handle the request by invoking the handler method, managing session
     * attributes lifecycle around the invocation.
     *
     * <h3>ch18 Enhancement:</h3>
     * The handle() method now wraps invocation with session attribute management:
     * <pre>
     *   1. Get/create SessionAttributesHandler for this controller
     *   2. Retrieve session attributes → store as request attribute
     *   3. Create SessionStatus → store as request attribute
     *   4. Resolve arguments (resolvers can find session attrs via request attribute)
     *   5. Invoke handler method
     *   6. Process return value → ModelAndView
     *   7. If SessionStatus.isComplete() → cleanup session attributes
     *      Else → store matching model attributes in session
     *   8. Return ModelAndView
     * </pre>
     *
     * Maps to: {@code RequestMappingHandlerAdapter.invokeHandlerMethod()} (line 885)
     * which creates ModelFactory, initializes the model (retrieves session attrs),
     * invokes the handler, and calls ModelFactory.updateModel() (stores/cleans up).
     */
    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response,
                               Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // ch18: Step 1 — Get or create SessionAttributesHandler for this controller
        SessionAttributesHandler sessionAttrHandler =
                getSessionAttributesHandler(handlerMethod);

        // ch18: Step 2 — Retrieve session attributes and store as request attribute
        // so ModelAttributeArgumentResolver can find existing instances
        Map<String, Object> sessionAttrs = sessionAttrHandler.retrieveAttributes(request);
        request.setAttribute(ModelAttributeArgumentResolver.SESSION_ATTRIBUTES_KEY, sessionAttrs);

        // ch18: Step 3 — Create per-request SessionStatus
        SimpleSessionStatus sessionStatus = new SimpleSessionStatus();
        request.setAttribute(SessionStatusArgumentResolver.SESSION_STATUS_ATTRIBUTE, sessionStatus);

        // Steps 4-6: Resolve arguments, invoke, process return value
        Object[] args = resolveArguments(handlerMethod, request);
        Object result = invokeHandlerMethod(handlerMethod, args);
        ModelAndView mv = returnValueHandlers.handleReturnValue(
                result, handlerMethod, request, response);

        // ch18: Step 7 — Session attributes lifecycle
        if (sessionAttrHandler.hasSessionAttributes()) {
            if (sessionStatus.isComplete()) {
                // SessionStatus.setComplete() was called — cleanup
                sessionAttrHandler.cleanupAttributes(request);
            } else {
                // Store matching model attributes back in session.
                // Two sources of attributes to store:
                // 1. Session attributes retrieved at start (may have been modified by reference)
                sessionAttrHandler.storeAttributes(request, sessionAttrs);
                // 2. New attributes in the ModelAndView model
                if (mv != null) {
                    sessionAttrHandler.storeAttributes(request, mv.getModel());
                }
            }
        }

        return mv;
    }

    /**
     * Get or create the SessionAttributesHandler for the given handler's controller class.
     *
     * Maps to: {@code RequestMappingHandlerAdapter.getSessionAttributesHandler()} (line 976)
     *
     * Cached per controller type so @SessionAttributes metadata is only
     * extracted once.
     */
    private SessionAttributesHandler getSessionAttributesHandler(HandlerMethod handlerMethod) {
        Class<?> handlerType = handlerMethod.getBeanType();
        return this.sessionAttributesHandlerCache.computeIfAbsent(
                handlerType,
                type -> new SessionAttributesHandler(type, this.sessionAttributeStore));
    }

    private Object[] resolveArguments(HandlerMethod handlerMethod, HttpServletRequest request) throws Exception {
        MethodParameter[] parameters = handlerMethod.getMethodParameters();
        if (parameters.length == 0) {
            return new Object[0];
        }
        Object[] args = new Object[parameters.length];
        for (int i = 0; i < parameters.length; i++) {
            MethodParameter param = parameters[i];
            if (argumentResolvers.supportsParameter(param)) {
                args[i] = argumentResolvers.resolveArgument(param, request);
            } else {
                throw new IllegalStateException(
                        "No suitable resolver for argument " + i + " of type '"
                                + param.getParameterType().getName()
                                + "' on method " + handlerMethod);
            }
        }
        return args;
    }

    private Object invokeHandlerMethod(HandlerMethod handlerMethod, Object[] args) throws Exception {
        try {
            return handlerMethod.getMethod().invoke(handlerMethod.getBean(), args);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof Exception e) {
                throw e;
            }
            throw ex;
        }
    }

    public HandlerMethodArgumentResolverComposite getArgumentResolvers() { return argumentResolvers; }
    public HandlerMethodReturnValueHandlerComposite getReturnValueHandlers() { return returnValueHandlers; }
    public List<HttpMessageConverter> getMessageConverters() { return messageConverters; }
    public ConversionService getConversionService() { return conversionService; }
    public ContentNegotiationManager getContentNegotiationManager() { return contentNegotiationManager; }

    /**
     * Returns the session attributes handler cache (for testing/inspection).
     *
     * <h3>ch18 Enhancement:</h3>
     * Added for testing session attribute management.
     */
    public Map<Class<?>, SessionAttributesHandler> getSessionAttributesHandlerCache() {
        return sessionAttributesHandlerCache;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` [MODIFIED]

```java
package com.simplespringmvc.servlet;

import com.simplespringmvc.adapter.HandlerAdapter;
import com.simplespringmvc.adapter.RedirectAttributesArgumentResolver;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.container.BeanContainer;
import com.simplespringmvc.exception.ExceptionHandlerExceptionResolver;
import com.simplespringmvc.exception.HandlerExceptionResolver;
import com.simplespringmvc.flash.FlashMap;
import com.simplespringmvc.flash.FlashMapManager;
import com.simplespringmvc.flash.SimpleRedirectAttributes;
import com.simplespringmvc.http.HttpMediaTypeNotAcceptableException;
import com.simplespringmvc.interceptor.HandlerExecutionChain;
import com.simplespringmvc.interceptor.HandlerInterceptor;
import com.simplespringmvc.mapping.HandlerMethod;
import com.simplespringmvc.mapping.SimpleHandlerMapping;
import com.simplespringmvc.view.ModelAndView;
import com.simplespringmvc.view.View;
import com.simplespringmvc.view.ViewResolver;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;

/**
 * The front controller that receives ALL HTTP requests and dispatches them
 * to the appropriate handler.
 *
 * Maps to: {@code org.springframework.web.servlet.DispatcherServlet}
 *
 * The real DispatcherServlet sits at the bottom of a three-class hierarchy:
 * <pre>
 *   HttpServletBean          → maps init-params to bean properties
 *     └── FrameworkServlet   → manages WebApplicationContext, request lifecycle
 *           └── DispatcherServlet → the actual dispatch logic
 * </pre>
 *
 * We collapse all three layers into one class. The key method is {@link #doDispatch}
 * which is the central routing point — every future feature adds a step here:
 * <ul>
 *   <li>ch03: handler mapping lookup</li>
 *   <li>ch04: handler adapter invocation</li>
 *   <li>ch11: interceptor pre/post processing</li>
 *   <li>ch12: exception handler resolution</li>
 *   <li>ch13: view resolution and rendering ✓</li>
 *   <li>ch18: flash map lifecycle and redirect handling ✓</li>
 * </ul>
 *
 * <h3>ch18 Enhancement:</h3>
 * The dispatch pipeline now includes flash map management and redirect support:
 * <pre>
 *   0. Flash map lifecycle: retrieve input flash map, create output flash map
 *   1. getHandler(request) → HandlerExecutionChain
 *   2. chain.applyPreHandle()
 *   3. ha.handle() → returns ModelAndView (null if response handled directly)
 *   4. chain.applyPostHandle()
 *   5. chain.triggerAfterCompletion() (always)
 *   6. If exception → processHandlerException()
 *   7. If ModelAndView non-null:
 *      7a. If view name starts with "redirect:" → handle redirect + save flash map
 *      7b. Otherwise → resolve and render view (merge input flash attrs into model)
 * </pre>
 *
 * Simplifications vs real Spring:
 * <ul>
 *   <li>No three-class hierarchy — one class does everything</li>
 *   <li>No WebApplicationContext — uses our simple BeanContainer</li>
 *   <li>No multipart handling</li>
 *   <li>No async/DeferredResult support</li>
 *   <li>No locale/theme resolution (added in ch20)</li>
 *   <li>Falls back to text/plain when view can't be resolved (real framework throws)</li>
 * </ul>
 */
public class SimpleDispatcherServlet extends HttpServlet {

    /**
     * Request attribute under which the input FlashMap is stored.
     *
     * Maps to: {@code DispatcherServlet.INPUT_FLASH_MAP_ATTRIBUTE} (line 221)
     */
    public static final String INPUT_FLASH_MAP_ATTRIBUTE =
            SimpleDispatcherServlet.class.getName() + ".INPUT_FLASH_MAP";

    /**
     * Request attribute under which the output FlashMap is stored.
     *
     * Maps to: {@code DispatcherServlet.OUTPUT_FLASH_MAP_ATTRIBUTE} (line 228)
     */
    public static final String OUTPUT_FLASH_MAP_ATTRIBUTE =
            SimpleDispatcherServlet.class.getName() + ".OUTPUT_FLASH_MAP";

    private final BeanContainer beanContainer;
    private SimpleHandlerMapping handlerMapping;
    private HandlerAdapter handlerAdapter;

    /**
     * Global interceptors applied to all handlers in order.
     *
     * Maps to: {@code AbstractHandlerMapping.adaptedInterceptors} (line 123)
     */
    private final List<HandlerInterceptor> interceptors = new ArrayList<>();

    /**
     * Exception resolvers consulted when a handler throws an exception.
     *
     * Maps to: {@code DispatcherServlet.handlerExceptionResolvers} (line 191)
     */
    private final List<HandlerExceptionResolver> exceptionResolvers = new ArrayList<>();

    /**
     * ViewResolvers that translate logical view names into View objects.
     *
     * Maps to: {@code DispatcherServlet.viewResolvers} (line 197)
     * Real version is populated from ViewResolver beans in the context
     * (or from DispatcherServlet.properties defaults). Iterated in order —
     * first resolver that returns a non-null View wins.
     *
     * <h3>ch13 Enhancement:</h3>
     * Added to support view resolution and rendering.
     */
    private final List<ViewResolver> viewResolvers = new ArrayList<>();

    /**
     * ch18: FlashMapManager for managing flash attributes across redirects.
     *
     * Maps to: {@code DispatcherServlet.flashMapManager} (line 200)
     * The real version is initialized from a FlashMapManager bean in the context,
     * defaulting to SessionFlashMapManager.
     */
    private FlashMapManager flashMapManager;

    public SimpleDispatcherServlet(BeanContainer beanContainer) {
        this.beanContainer = beanContainer;
    }

    /**
     * Called by Tomcat once the Servlet is registered. Initializes strategy
     * components (handler mappings, adapters, etc.) from the bean container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
     */
    @Override
    public void init() throws ServletException {
        initStrategies();
    }

    /**
     * Initialize all strategy beans from the container.
     *
     * Maps to: {@code DispatcherServlet.initStrategies()} (line 441)
     * Real version initializes 9 strategy types. We initialize them
     * incrementally as features are added.
     */
    protected void initStrategies() {
        initHandlerMapping();
        initHandlerAdapter();
        initExceptionResolvers();
        // ch13: ViewResolvers are added programmatically via addViewResolver()
        // rather than initialized from the container. The real framework finds
        // ViewResolver beans in the ApplicationContext.
        // ch18: FlashMapManager is set programmatically via setFlashMapManager()
    }

    private void initHandlerMapping() {
        handlerMapping = new SimpleHandlerMapping();
        handlerMapping.init(beanContainer);
    }

    private void initHandlerAdapter() {
        handlerAdapter = new SimpleHandlerAdapter();
    }

    private void initExceptionResolvers() {
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.init(beanContainer);
        exceptionResolvers.add(resolver);
    }

    /**
     * Override service() to route ALL HTTP methods through doDispatch().
     *
     * Maps to: {@code FrameworkServlet.service()} (line 870)
     */
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            throw new ServletException("Dispatch failed", ex);
        }
    }

    /**
     * The central dispatch method — the heart of the framework.
     *
     * Maps to: {@code DispatcherServlet.doDispatch()} (line 935)
     *
     * <h3>ch18 Enhancement:</h3>
     * The dispatch flow now includes flash map management:
     * <pre>
     *   0. Flash map lifecycle:
     *      - Retrieve input flash map (from previous redirect)
     *      - Create output flash map (for possible redirect in this request)
     *   1. mappedHandler = getHandler(request)
     *   2. mappedHandler.applyPreHandle()
     *   3. mv = ha.handle(request, response, handler) ← now manages session attrs
     *   4. mappedHandler.applyPostHandle()
     *   5. mappedHandler.triggerAfterCompletion() (always)
     *   6. processHandlerException() if exception
     *   7. If view name starts with "redirect:" → handleRedirect() + save flash map
     *      Otherwise → render view (merge input flash attrs into model)
     * </pre>
     *
     * The flash map lifecycle mirrors the real DispatcherServlet.doService()
     * (lines 850-857) which sets up INPUT/OUTPUT flash maps as request attributes
     * before doDispatch() is called.
     */
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        HandlerExecutionChain mappedHandler = null;
        ModelAndView mv = null;
        Exception dispatchException = null;

        // ch18: Step 0 — Flash map lifecycle: retrieve input, create output
        FlashMap inputFlashMap = null;
        if (flashMapManager != null) {
            inputFlashMap = flashMapManager.retrieveAndUpdate(request, response);
            if (inputFlashMap != null) {
                request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE,
                        Collections.unmodifiableMap(inputFlashMap));
            }
            request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
        }

        try {
            // Step 1: Look up the handler + interceptor chain for this request
            mappedHandler = getHandler(request);

            if (mappedHandler == null) {
                response.sendError(HttpServletResponse.SC_NOT_FOUND,
                        "No handler found for " + request.getMethod() + " " + request.getRequestURI());
                return;
            }

            // Step 2: Apply interceptor preHandle — forward order
            if (!mappedHandler.applyPreHandle(request, response)) {
                return;
            }

            // Step 3: Find a HandlerAdapter and invoke the handler.
            // ch18: handle() now manages session attributes lifecycle internally.
            HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler());
            mv = adapter.handle(request, response, mappedHandler.getHandler());

            // Step 4: Apply interceptor postHandle — reverse order
            mappedHandler.applyPostHandle(request, response);

        } catch (HttpMediaTypeNotAcceptableException ex) {
            // ch14: The client's Accept header doesn't match any type the server can produce.
            response.sendError(HttpServletResponse.SC_NOT_ACCEPTABLE, ex.getMessage());
            if (mappedHandler != null) {
                mappedHandler.triggerAfterCompletion(request, response, null);
            }
            return;
        } catch (Exception ex) {
            dispatchException = ex;
        }

        // Step 5: afterCompletion — always runs
        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(request, response, dispatchException);
        }

        // Step 6: Exception handling
        if (dispatchException != null) {
            Object handler = (mappedHandler != null) ? mappedHandler.getHandler() : null;
            if (!processHandlerException(request, response, handler, dispatchException)) {
                throw dispatchException;
            }
            // Exception was handled — don't render a view
            return;
        }

        // Step 7 (ch13/ch18): View resolution, rendering, or redirect.
        if (mv != null && mv.getViewName() != null) {
            String viewName = mv.getViewName();

            // ch18: Handle "redirect:" prefix
            if (viewName.startsWith("redirect:")) {
                String redirectUrl = viewName.substring("redirect:".length());
                handleRedirect(redirectUrl, request, response);
            } else {
                // ch18: Merge input flash attributes into model for view access
                if (inputFlashMap != null && !inputFlashMap.isEmpty()) {
                    mv.getModel().putAll(inputFlashMap);
                }
                render(mv, request, response);
            }
        }
    }

    /**
     * Handle a redirect by saving flash attributes and sending the redirect response.
     *
     * <h3>ch18 Enhancement:</h3>
     * Maps to the redirect handling in {@code DispatcherServlet.processDispatchResult()}
     * and {@code RequestContextUtils.saveOutputFlashMap()} (line 218).
     *
     * The flow:
     * <ol>
     *   <li>Get the output FlashMap (created at request start)</li>
     *   <li>Copy flash attributes from RedirectAttributes (if a handler used it)</li>
     *   <li>Set the target request path on the FlashMap</li>
     *   <li>Save the FlashMap via FlashMapManager</li>
     *   <li>Send the HTTP redirect response</li>
     * </ol>
     *
     * @param redirectUrl the target URL to redirect to
     * @param request     current HTTP request
     * @param response    current HTTP response
     */
    private void handleRedirect(String redirectUrl, HttpServletRequest request,
                                 HttpServletResponse response) throws IOException {
        if (flashMapManager != null) {
            FlashMap outputFlashMap =
                    (FlashMap) request.getAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE);

            // Copy flash attributes from RedirectAttributes if the handler used it
            SimpleRedirectAttributes redirectAttrs = (SimpleRedirectAttributes)
                    request.getAttribute(RedirectAttributesArgumentResolver.REDIRECT_ATTRIBUTES_ATTRIBUTE);
            if (redirectAttrs != null && !redirectAttrs.getFlashAttributes().isEmpty()) {
                outputFlashMap.putAll(redirectAttrs.getFlashAttributes());
            }

            // Set target path so the FlashMap matches the redirect target
            outputFlashMap.setTargetRequestPath(redirectUrl);

            // Save to session for the next request to retrieve
            flashMapManager.saveOutputFlashMap(outputFlashMap, request, response);
        }

        response.sendRedirect(redirectUrl);
    }

    /**
     * Resolve the view name and render the view with model data.
     *
     * Maps to: {@code DispatcherServlet.render()} (line 1267)
     *
     * The real render() method:
     * <pre>
     *   1. Determine locale (via LocaleResolver)
     *   2. Resolve view name → View object (via ViewResolver chain)
     *   3. Set response status if configured
     *   4. Call view.render(model, request, response)
     * </pre>
     *
     * We skip step 1 (no locale support yet — ch20) and step 3 (no status code
     * on ModelAndView). If no ViewResolver resolves the view name, we fall back
     * to writing the view name as text/plain. This is a simplification — the
     * real framework throws a {@code ServletException} when a view can't be
     * resolved.
     *
     * @param mv       the ModelAndView holding view name and model data
     * @param request  current HTTP request
     * @param response current HTTP response
     */
    private void render(ModelAndView mv, HttpServletRequest request,
                        HttpServletResponse response) throws Exception {
        String viewName = mv.getViewName();

        // Try to resolve the view name via the ViewResolver chain.
        // Maps to: DispatcherServlet.resolveViewName() (line 1339)
        View view = resolveViewName(viewName);

        if (view != null) {
            // Delegate to the View object for rendering.
            // Maps to: DispatcherServlet.render() (line 1305):
            //   view.render(mv.getModelInternal(), request, response)
            view.render(mv.getModel(), request, response);
        } else {
            // Fallback: write the view name directly as text/plain.
            // This is a simplification — the real framework throws:
            //   throw new ServletException("Could not resolve view with name '" + viewName + "'");
            // We fall back gracefully to preserve backward compatibility with
            // String-returning controllers that don't have templates yet.
            response.setContentType("text/plain");
            response.setCharacterEncoding("UTF-8");
            response.getWriter().write(viewName);
            response.getWriter().flush();
        }
    }

    /**
     * Resolve a view name to a View object by iterating the ViewResolver chain.
     *
     * Maps to: {@code DispatcherServlet.resolveViewName()} (line 1339)
     * → iterates {@code viewResolvers} and returns the first non-null result.
     *
     * @param viewName the logical view name to resolve
     * @return the View object, or null if no resolver can resolve it
     */
    private View resolveViewName(String viewName) throws Exception {
        for (ViewResolver resolver : viewResolvers) {
            View view = resolver.resolveViewName(viewName);
            if (view != null) {
                return view;
            }
        }
        return null;
    }

    /**
     * Iterate exception resolvers to handle the given exception.
     *
     * Maps to: {@code DispatcherServlet.processHandlerException()} (line 1207)
     */
    private boolean processHandlerException(HttpServletRequest request, HttpServletResponse response,
                                            Object handler, Exception ex) {
        for (HandlerExceptionResolver resolver : exceptionResolvers) {
            if (resolver.resolveException(request, response, handler, ex)) {
                return true;
            }
        }
        return false;
    }

    private HandlerExecutionChain getHandler(HttpServletRequest request) {
        if (handlerMapping == null) {
            return null;
        }
        HandlerMethod handler = handlerMapping.lookupHandler(request);
        if (handler == null) {
            return null;
        }
        return new HandlerExecutionChain(handler, interceptors);
    }

    private HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
        if (handlerAdapter != null && handlerAdapter.supports(handler)) {
            return handlerAdapter;
        }
        throw new ServletException(
                "No adapter for handler [" + handler + "]: does the DispatcherServlet "
                        + "configuration include a HandlerAdapter that supports this handler?");
    }

    // ─── FlashMapManager registration ────────────────────────────────

    /**
     * Set the FlashMapManager for managing flash attributes across redirects.
     *
     * Maps to: In the real framework, the FlashMapManager is initialized from
     * a FlashMapManager bean in the ApplicationContext (defaulting to
     * SessionFlashMapManager). We use programmatic registration for simplicity.
     *
     * <h3>ch18 Enhancement:</h3>
     * Added to support flash attributes and the Post/Redirect/Get pattern.
     *
     * @param flashMapManager the manager to use
     */
    public void setFlashMapManager(FlashMapManager flashMapManager) {
        this.flashMapManager = flashMapManager;
    }

    /**
     * Returns the configured FlashMapManager (for testing/inspection).
     */
    public FlashMapManager getFlashMapManager() {
        return flashMapManager;
    }

    // ─── ViewResolver registration ──────────────────────────────────

    /**
     * Register a ViewResolver to resolve view names to View objects.
     *
     * Maps to: In the real framework, ViewResolvers are registered as beans in
     * the ApplicationContext and auto-detected by DispatcherServlet.initViewResolvers().
     * We use programmatic registration for simplicity.
     *
     * <h3>ch13 Enhancement:</h3>
     * Added to support view resolution.
     *
     * @param viewResolver the resolver to add
     */
    public void addViewResolver(ViewResolver viewResolver) {
        this.viewResolvers.add(viewResolver);
    }

    /**
     * Returns the registered ViewResolvers (for testing/inspection).
     */
    public List<ViewResolver> getViewResolvers() {
        return List.copyOf(viewResolvers);
    }

    // ─── Interceptor registration ────────────────────────────────────

    public void addInterceptor(HandlerInterceptor interceptor) {
        this.interceptors.add(interceptor);
    }

    public List<HandlerInterceptor> getInterceptors() {
        return List.copyOf(interceptors);
    }

    // ─── Accessors ───────────────────────────────────────────────────

    public BeanContainer getBeanContainer() {
        return beanContainer;
    }

    public SimpleHandlerMapping getHandlerMapping() {
        return handlerMapping;
    }

    public HandlerAdapter getHandlerAdapter() {
        return handlerAdapter;
    }

    public List<HandlerExceptionResolver> getExceptionResolvers() {
        return List.copyOf(exceptionResolvers);
    }
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/session/SimpleSessionStatusTest.java` [NEW]

```java
package com.simplespringmvc.session;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Unit tests for {@link SimpleSessionStatus}.
 */
class SimpleSessionStatusTest {

    @Test
    @DisplayName("should not be complete initially")
    void shouldNotBeCompleteInitially() {
        var status = new SimpleSessionStatus();
        assertThat(status.isComplete()).isFalse();
    }

    @Test
    @DisplayName("should be complete after setComplete")
    void shouldBeComplete_AfterSetComplete() {
        var status = new SimpleSessionStatus();
        status.setComplete();
        assertThat(status.isComplete()).isTrue();
    }
}
```

#### File: `src/test/java/com/simplespringmvc/session/DefaultSessionAttributeStoreTest.java` [NEW]

```java
package com.simplespringmvc.session;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link DefaultSessionAttributeStore}.
 */
class DefaultSessionAttributeStoreTest {

    private final DefaultSessionAttributeStore store = new DefaultSessionAttributeStore();

    @Test
    @DisplayName("should store attribute in HttpSession")
    void shouldStoreAttribute() {
        HttpServletRequest request = mock(HttpServletRequest.class);
        HttpSession session = mock(HttpSession.class);
        when(request.getSession()).thenReturn(session);

        store.storeAttribute(request, "user", "John");

        verify(session).setAttribute("user", "John");
    }

    @Test
    @DisplayName("should retrieve attribute from HttpSession")
    void shouldRetrieveAttribute() {
        HttpServletRequest request = mock(HttpServletRequest.class);
        HttpSession session = mock(HttpSession.class);
        when(request.getSession(false)).thenReturn(session);
        when(session.getAttribute("user")).thenReturn("John");

        Object result = store.retrieveAttribute(request, "user");

        assertThat(result).isEqualTo("John");
    }

    @Test
    @DisplayName("should return null when no session exists")
    void shouldReturnNull_WhenNoSession() {
        HttpServletRequest request = mock(HttpServletRequest.class);
        when(request.getSession(false)).thenReturn(null);

        Object result = store.retrieveAttribute(request, "user");

        assertThat(result).isNull();
    }

    @Test
    @DisplayName("should cleanup attribute from HttpSession")
    void shouldCleanupAttribute() {
        HttpServletRequest request = mock(HttpServletRequest.class);
        HttpSession session = mock(HttpSession.class);
        when(request.getSession(false)).thenReturn(session);

        store.cleanupAttribute(request, "user");

        verify(session).removeAttribute("user");
    }

    @Test
    @DisplayName("should not fail when cleaning up with no session")
    void shouldNotFail_WhenCleaningUpWithNoSession() {
        HttpServletRequest request = mock(HttpServletRequest.class);
        when(request.getSession(false)).thenReturn(null);

        store.cleanupAttribute(request, "user"); // should not throw
    }
}
```

#### File: `src/test/java/com/simplespringmvc/session/SessionAttributesHandlerTest.java` [NEW]

```java
package com.simplespringmvc.session;

import com.simplespringmvc.annotation.SessionAttributes;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link SessionAttributesHandler}.
 */
class SessionAttributesHandlerTest {

    // ─── Test controllers ────────────────────────────────────────────

    @SessionAttributes({"user", "cart"})
    static class NameBasedController {}

    @SessionAttributes(types = {String.class, Integer.class})
    static class TypeBasedController {}

    @SessionAttributes(value = "user", types = Double.class)
    static class MixedController {}

    static class NoSessionAttributesController {}

    // ─── Test helpers ────────────────────────────────────────────────

    private HttpServletRequest mockRequest() {
        HttpServletRequest request = mock(HttpServletRequest.class);
        HttpSession session = mock(HttpSession.class);
        when(request.getSession()).thenReturn(session);
        when(request.getSession(false)).thenReturn(session);
        return request;
    }

    // ─── Tests ───────────────────────────────────────────────────────

    @Nested
    @DisplayName("hasSessionAttributes")
    class HasSessionAttributes {

        @Test
        @DisplayName("should return true when @SessionAttributes declares names")
        void shouldReturnTrue_WhenNamesDeclared() {
            var handler = new SessionAttributesHandler(
                    NameBasedController.class, new DefaultSessionAttributeStore());
            assertThat(handler.hasSessionAttributes()).isTrue();
        }

        @Test
        @DisplayName("should return true when @SessionAttributes declares types")
        void shouldReturnTrue_WhenTypesDeclared() {
            var handler = new SessionAttributesHandler(
                    TypeBasedController.class, new DefaultSessionAttributeStore());
            assertThat(handler.hasSessionAttributes()).isTrue();
        }

        @Test
        @DisplayName("should return false when @SessionAttributes is absent")
        void shouldReturnFalse_WhenAnnotationAbsent() {
            var handler = new SessionAttributesHandler(
                    NoSessionAttributesController.class, new DefaultSessionAttributeStore());
            assertThat(handler.hasSessionAttributes()).isFalse();
        }
    }

    @Nested
    @DisplayName("isHandlerSessionAttribute")
    class IsHandlerSessionAttribute {

        @Test
        @DisplayName("should match attribute by name")
        void shouldMatchByName() {
            var handler = new SessionAttributesHandler(
                    NameBasedController.class, new DefaultSessionAttributeStore());
            assertThat(handler.isHandlerSessionAttribute("user", Object.class)).isTrue();
            assertThat(handler.isHandlerSessionAttribute("cart", Object.class)).isTrue();
            assertThat(handler.isHandlerSessionAttribute("unknown", Object.class)).isFalse();
        }

        @Test
        @DisplayName("should match attribute by type")
        void shouldMatchByType() {
            var handler = new SessionAttributesHandler(
                    TypeBasedController.class, new DefaultSessionAttributeStore());
            assertThat(handler.isHandlerSessionAttribute("anything", String.class)).isTrue();
            assertThat(handler.isHandlerSessionAttribute("number", Integer.class)).isTrue();
            assertThat(handler.isHandlerSessionAttribute("nope", Double.class)).isFalse();
        }

        @Test
        @DisplayName("should add type-matched attribute name to knownAttributeNames")
        void shouldTrackTypeMatchedNames() {
            var handler = new SessionAttributesHandler(
                    TypeBasedController.class, new DefaultSessionAttributeStore());
            handler.isHandlerSessionAttribute("myString", String.class);
            assertThat(handler.getKnownAttributeNames()).contains("myString");
        }

        @Test
        @DisplayName("should match by name OR type in mixed mode")
        void shouldMatchNameOrType_WhenMixed() {
            var handler = new SessionAttributesHandler(
                    MixedController.class, new DefaultSessionAttributeStore());
            // Match by name
            assertThat(handler.isHandlerSessionAttribute("user", Object.class)).isTrue();
            // Match by type
            assertThat(handler.isHandlerSessionAttribute("score", Double.class)).isTrue();
            // No match
            assertThat(handler.isHandlerSessionAttribute("unknown", Long.class)).isFalse();
        }
    }

    @Nested
    @DisplayName("storeAttributes")
    class StoreAttributes {

        @Test
        @DisplayName("should store matching attributes in session")
        void shouldStoreMatchingAttributes() {
            SessionAttributeStore store = mock(SessionAttributeStore.class);
            var handler = new SessionAttributesHandler(NameBasedController.class, store);

            Map<String, Object> model = Map.of(
                    "user", "John",
                    "cart", 42,
                    "unrelated", "skip"
            );

            HttpServletRequest request = mockRequest();
            handler.storeAttributes(request, model);

            verify(store).storeAttribute(request, "user", "John");
            verify(store).storeAttribute(request, "cart", 42);
            verify(store, never()).storeAttribute(eq(request), eq("unrelated"), any());
        }

        @Test
        @DisplayName("should skip null values")
        void shouldSkipNullValues() {
            SessionAttributeStore store = mock(SessionAttributeStore.class);
            var handler = new SessionAttributesHandler(NameBasedController.class, store);

            Map<String, Object> model = new java.util.HashMap<>();
            model.put("user", null);

            HttpServletRequest request = mockRequest();
            handler.storeAttributes(request, model);

            verify(store, never()).storeAttribute(any(), any(), any());
        }
    }

    @Nested
    @DisplayName("retrieveAttributes")
    class RetrieveAttributes {

        @Test
        @DisplayName("should retrieve known attributes from session")
        void shouldRetrieveKnownAttributes() {
            SessionAttributeStore store = mock(SessionAttributeStore.class);
            var handler = new SessionAttributesHandler(NameBasedController.class, store);

            HttpServletRequest request = mockRequest();
            when(store.retrieveAttribute(request, "user")).thenReturn("John");
            when(store.retrieveAttribute(request, "cart")).thenReturn(42);

            Map<String, Object> result = handler.retrieveAttributes(request);

            assertThat(result).containsEntry("user", "John");
            assertThat(result).containsEntry("cart", 42);
        }

        @Test
        @DisplayName("should exclude null values from result")
        void shouldExcludeNullValues() {
            SessionAttributeStore store = mock(SessionAttributeStore.class);
            var handler = new SessionAttributesHandler(NameBasedController.class, store);

            HttpServletRequest request = mockRequest();
            when(store.retrieveAttribute(request, "user")).thenReturn("John");
            when(store.retrieveAttribute(request, "cart")).thenReturn(null);

            Map<String, Object> result = handler.retrieveAttributes(request);

            assertThat(result).containsKey("user");
            assertThat(result).doesNotContainKey("cart");
        }
    }

    @Nested
    @DisplayName("cleanupAttributes")
    class CleanupAttributes {

        @Test
        @DisplayName("should cleanup all known attributes from session")
        void shouldCleanupAllKnownAttributes() {
            SessionAttributeStore store = mock(SessionAttributeStore.class);
            var handler = new SessionAttributesHandler(NameBasedController.class, store);

            HttpServletRequest request = mockRequest();
            handler.cleanupAttributes(request);

            verify(store).cleanupAttribute(request, "user");
            verify(store).cleanupAttribute(request, "cart");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/flash/FlashMapTest.java` [NEW]

```java
package com.simplespringmvc.flash;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Unit tests for {@link FlashMap}.
 */
class FlashMapTest {

    @Nested
    @DisplayName("Flash attribute storage")
    class FlashAttributeStorage {

        @Test
        @DisplayName("should store and retrieve flash attributes")
        void shouldStoreAndRetrieve() {
            FlashMap flashMap = new FlashMap();
            flashMap.put("message", "Success!");
            flashMap.put("count", 42);

            assertThat(flashMap.get("message")).isEqualTo("Success!");
            assertThat(flashMap.get("count")).isEqualTo(42);
        }

        @Test
        @DisplayName("should be empty initially")
        void shouldBeEmptyInitially() {
            FlashMap flashMap = new FlashMap();
            assertThat(flashMap).isEmpty();
        }
    }

    @Nested
    @DisplayName("Target request path")
    class TargetRequestPath {

        @Test
        @DisplayName("should store and retrieve target request path")
        void shouldStoreTargetPath() {
            FlashMap flashMap = new FlashMap();
            flashMap.setTargetRequestPath("/order/confirmation");

            assertThat(flashMap.getTargetRequestPath()).isEqualTo("/order/confirmation");
        }

        @Test
        @DisplayName("should return null when no target path set")
        void shouldReturnNull_WhenNoTargetPathSet() {
            FlashMap flashMap = new FlashMap();
            assertThat(flashMap.getTargetRequestPath()).isNull();
        }
    }

    @Nested
    @DisplayName("Expiration")
    class Expiration {

        @Test
        @DisplayName("should not be expired initially")
        void shouldNotBeExpiredInitially() {
            FlashMap flashMap = new FlashMap();
            assertThat(flashMap.isExpired()).isFalse();
        }

        @Test
        @DisplayName("should not be expired when TTL is long")
        void shouldNotBeExpired_WhenTtlIsLong() {
            FlashMap flashMap = new FlashMap();
            flashMap.startExpirationPeriod(3600); // 1 hour
            assertThat(flashMap.isExpired()).isFalse();
        }

        @Test
        @DisplayName("should be expired when TTL is zero")
        void shouldBeExpired_WhenTtlIsZero() throws InterruptedException {
            FlashMap flashMap = new FlashMap();
            flashMap.startExpirationPeriod(0);
            // TTL=0 means expires immediately (currentTime > expirationTime since expirationTime = now)
            Thread.sleep(5); // tiny sleep to ensure time passes
            assertThat(flashMap.isExpired()).isTrue();
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/flash/SessionFlashMapManagerTest.java` [NEW]

```java
package com.simplespringmvc.flash;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link SessionFlashMapManager}.
 */
class SessionFlashMapManagerTest {

    private static final String FLASH_MAPS_KEY =
            SessionFlashMapManager.class.getName() + ".FLASH_MAPS";

    private SessionFlashMapManager manager;
    private HttpServletRequest request;
    private HttpServletResponse response;
    private HttpSession session;

    @BeforeEach
    void setUp() {
        manager = new SessionFlashMapManager();
        request = mock(HttpServletRequest.class);
        response = mock(HttpServletResponse.class);
        session = mock(HttpSession.class);
        when(request.getSession()).thenReturn(session);
        when(request.getSession(false)).thenReturn(session);
    }

    @Nested
    @DisplayName("saveOutputFlashMap")
    class SaveOutputFlashMap {

        @Test
        @DisplayName("should save non-empty flash map to session")
        void shouldSaveFlashMap() {
            FlashMap flashMap = new FlashMap();
            flashMap.put("message", "Saved!");
            flashMap.setTargetRequestPath("/target");

            when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(null);

            manager.saveOutputFlashMap(flashMap, request, response);

            verify(session).setAttribute(eq(FLASH_MAPS_KEY), argThat(arg -> {
                @SuppressWarnings("unchecked")
                List<FlashMap> maps = (List<FlashMap>) arg;
                return maps.size() == 1 && maps.get(0).containsKey("message");
            }));
        }

        @Test
        @DisplayName("should not save empty flash map")
        void shouldNotSaveEmptyFlashMap() {
            FlashMap flashMap = new FlashMap(); // empty

            manager.saveOutputFlashMap(flashMap, request, response);

            verify(session, never()).setAttribute(eq(FLASH_MAPS_KEY), any());
        }

        @Test
        @DisplayName("should append to existing flash maps")
        void shouldAppendToExisting() {
            FlashMap existing = new FlashMap();
            existing.put("old", "data");
            existing.startExpirationPeriod(180);
            List<FlashMap> existingMaps = new ArrayList<>();
            existingMaps.add(existing);

            when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(existingMaps);

            FlashMap newFlashMap = new FlashMap();
            newFlashMap.put("new", "data");

            manager.saveOutputFlashMap(newFlashMap, request, response);

            verify(session).setAttribute(eq(FLASH_MAPS_KEY), argThat(arg -> {
                @SuppressWarnings("unchecked")
                List<FlashMap> maps = (List<FlashMap>) arg;
                return maps.size() == 2;
            }));
        }
    }

    @Nested
    @DisplayName("retrieveAndUpdate")
    class RetrieveAndUpdate {

        @Test
        @DisplayName("should return null when no flash maps in session")
        void shouldReturnNull_WhenNoFlashMaps() {
            when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(null);

            FlashMap result = manager.retrieveAndUpdate(request, response);

            assertThat(result).isNull();
        }

        @Test
        @DisplayName("should return matching flash map by target path")
        void shouldReturnMatchingFlashMap() {
            FlashMap flashMap = new FlashMap();
            flashMap.put("message", "Hello!");
            flashMap.setTargetRequestPath("/target");
            flashMap.startExpirationPeriod(180);
            List<FlashMap> maps = new ArrayList<>();
            maps.add(flashMap);

            when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(maps);
            when(request.getRequestURI()).thenReturn("/target");

            FlashMap result = manager.retrieveAndUpdate(request, response);

            assertThat(result).isNotNull();
            assertThat(result.get("message")).isEqualTo("Hello!");
        }

        @Test
        @DisplayName("should not return flash map with wrong target path")
        void shouldNotReturnWrongPath() {
            FlashMap flashMap = new FlashMap();
            flashMap.put("message", "Hello!");
            flashMap.setTargetRequestPath("/other");
            flashMap.startExpirationPeriod(180);
            List<FlashMap> maps = new ArrayList<>();
            maps.add(flashMap);

            when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(maps);
            when(request.getRequestURI()).thenReturn("/target");

            FlashMap result = manager.retrieveAndUpdate(request, response);

            assertThat(result).isNull();
        }

        @Test
        @DisplayName("should return flash map with no target path (matches any request)")
        void shouldReturnFlashMapWithNoTargetPath() {
            FlashMap flashMap = new FlashMap();
            flashMap.put("message", "Hello!");
            // no target path set — matches any request
            flashMap.startExpirationPeriod(180);
            List<FlashMap> maps = new ArrayList<>();
            maps.add(flashMap);

            when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(maps);
            when(request.getRequestURI()).thenReturn("/anything");

            FlashMap result = manager.retrieveAndUpdate(request, response);

            assertThat(result).isNotNull();
            assertThat(result.get("message")).isEqualTo("Hello!");
        }

        @Test
        @DisplayName("should remove consumed flash map from session")
        void shouldRemoveConsumedFlashMap() {
            FlashMap flashMap = new FlashMap();
            flashMap.put("message", "Hello!");
            flashMap.setTargetRequestPath("/target");
            flashMap.startExpirationPeriod(180);
            List<FlashMap> maps = new ArrayList<>();
            maps.add(flashMap);

            when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(maps);
            when(request.getRequestURI()).thenReturn("/target");

            manager.retrieveAndUpdate(request, response);

            // After consuming the only flash map, session attr should be removed
            verify(session).removeAttribute(FLASH_MAPS_KEY);
        }

        @Test
        @DisplayName("should remove expired flash maps during retrieval")
        void shouldRemoveExpiredFlashMaps() {
            FlashMap expiredMap = new FlashMap();
            expiredMap.put("old", "data");
            expiredMap.startExpirationPeriod(0); // expires immediately

            FlashMap validMap = new FlashMap();
            validMap.put("new", "data");
            validMap.setTargetRequestPath("/target");
            validMap.startExpirationPeriod(180);

            List<FlashMap> maps = new ArrayList<>();
            maps.add(expiredMap);
            maps.add(validMap);

            // Small sleep to ensure the expired map is actually expired
            try { Thread.sleep(5); } catch (InterruptedException ignored) {}

            when(session.getAttribute(FLASH_MAPS_KEY)).thenReturn(maps);
            when(request.getRequestURI()).thenReturn("/target");

            FlashMap result = manager.retrieveAndUpdate(request, response);

            assertThat(result).isNotNull();
            assertThat(result.get("new")).isEqualTo("data");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/flash/SimpleRedirectAttributesTest.java` [NEW]

```java
package com.simplespringmvc.flash;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Unit tests for {@link SimpleRedirectAttributes}.
 */
class SimpleRedirectAttributesTest {

    @Test
    @DisplayName("should store flash attributes")
    void shouldStoreFlashAttributes() {
        var attrs = new SimpleRedirectAttributes();
        attrs.addFlashAttribute("message", "Saved!");
        attrs.addFlashAttribute("count", 3);

        assertThat(attrs.getFlashAttributes())
                .containsEntry("message", "Saved!")
                .containsEntry("count", 3);
    }

    @Test
    @DisplayName("should be empty initially")
    void shouldBeEmptyInitially() {
        var attrs = new SimpleRedirectAttributes();
        assertThat(attrs.getFlashAttributes()).isEmpty();
    }

    @Test
    @DisplayName("should support fluent chaining")
    void shouldSupportFluentChaining() {
        var attrs = new SimpleRedirectAttributes()
                .addFlashAttribute("a", 1)
                .addFlashAttribute("b", 2);

        assertThat(attrs.getFlashAttributes()).hasSize(2);
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/SessionAttributeArgumentResolverTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.SessionAttribute;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link SessionAttributeArgumentResolver}.
 */
class SessionAttributeArgumentResolverTest {

    private final SessionAttributeArgumentResolver resolver = new SessionAttributeArgumentResolver();

    // ─── Test methods ────────────────────────────────────────────────

    @SuppressWarnings("unused")
    static void withSessionAttribute(@SessionAttribute("currentUser") String user) {}

    @SuppressWarnings("unused")
    static void withOptionalSessionAttribute(@SessionAttribute(value = "token", required = false) String token) {}

    @SuppressWarnings("unused")
    static void withDefaultName(@SessionAttribute String myAttr) {}

    @SuppressWarnings("unused")
    static void withoutAnnotation(String plain) {}

    // ─── Tests ───────────────────────────────────────────────────────

    @Test
    @DisplayName("should support parameter with @SessionAttribute")
    void shouldSupportAnnotatedParameter() throws Exception {
        Method method = getClass().getDeclaredMethod("withSessionAttribute", String.class);
        MethodParameter param = new MethodParameter(method, 0);

        assertThat(resolver.supportsParameter(param)).isTrue();
    }

    @Test
    @DisplayName("should not support parameter without @SessionAttribute")
    void shouldNotSupportUnannotatedParameter() throws Exception {
        Method method = getClass().getDeclaredMethod("withoutAnnotation", String.class);
        MethodParameter param = new MethodParameter(method, 0);

        assertThat(resolver.supportsParameter(param)).isFalse();
    }

    @Test
    @DisplayName("should resolve attribute from session by explicit name")
    void shouldResolveByExplicitName() throws Exception {
        Method method = getClass().getDeclaredMethod("withSessionAttribute", String.class);
        MethodParameter param = new MethodParameter(method, 0);

        HttpServletRequest request = mock(HttpServletRequest.class);
        HttpSession session = mock(HttpSession.class);
        when(request.getSession(false)).thenReturn(session);
        when(session.getAttribute("currentUser")).thenReturn("John");

        Object result = resolver.resolveArgument(param, request);

        assertThat(result).isEqualTo("John");
    }

    @Test
    @DisplayName("should throw when required attribute is missing")
    void shouldThrow_WhenRequiredAttributeMissing() throws Exception {
        Method method = getClass().getDeclaredMethod("withSessionAttribute", String.class);
        MethodParameter param = new MethodParameter(method, 0);

        HttpServletRequest request = mock(HttpServletRequest.class);
        when(request.getSession(false)).thenReturn(null);

        assertThatThrownBy(() -> resolver.resolveArgument(param, request))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("currentUser");
    }

    @Test
    @DisplayName("should return null when optional attribute is missing")
    void shouldReturnNull_WhenOptionalAttributeMissing() throws Exception {
        Method method = getClass().getDeclaredMethod("withOptionalSessionAttribute", String.class);
        MethodParameter param = new MethodParameter(method, 0);

        HttpServletRequest request = mock(HttpServletRequest.class);
        when(request.getSession(false)).thenReturn(null);

        Object result = resolver.resolveArgument(param, request);

        assertThat(result).isNull();
    }

    @Test
    @DisplayName("should use parameter name when @SessionAttribute value is empty")
    void shouldUseParameterName_WhenValueEmpty() throws Exception {
        Method method = getClass().getDeclaredMethod("withDefaultName", String.class);
        MethodParameter param = new MethodParameter(method, 0);

        HttpServletRequest request = mock(HttpServletRequest.class);
        HttpSession session = mock(HttpSession.class);
        when(request.getSession(false)).thenReturn(session);
        when(session.getAttribute("myAttr")).thenReturn("value");

        Object result = resolver.resolveArgument(param, request);

        assertThat(result).isEqualTo("value");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/SessionStatusArgumentResolverTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.session.SessionStatus;
import com.simplespringmvc.session.SimpleSessionStatus;
import jakarta.servlet.http.HttpServletRequest;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link SessionStatusArgumentResolver}.
 */
class SessionStatusArgumentResolverTest {

    private final SessionStatusArgumentResolver resolver = new SessionStatusArgumentResolver();

    // ─── Test methods ────────────────────────────────────────────────

    @SuppressWarnings("unused")
    static void withSessionStatus(SessionStatus status) {}

    @SuppressWarnings("unused")
    static void withString(String plain) {}

    // ─── Tests ───────────────────────────────────────────────────────

    @Test
    @DisplayName("should support SessionStatus parameter")
    void shouldSupportSessionStatus() throws Exception {
        Method method = getClass().getDeclaredMethod("withSessionStatus", SessionStatus.class);
        MethodParameter param = new MethodParameter(method, 0);

        assertThat(resolver.supportsParameter(param)).isTrue();
    }

    @Test
    @DisplayName("should not support non-SessionStatus parameter")
    void shouldNotSupportOtherTypes() throws Exception {
        Method method = getClass().getDeclaredMethod("withString", String.class);
        MethodParameter param = new MethodParameter(method, 0);

        assertThat(resolver.supportsParameter(param)).isFalse();
    }

    @Test
    @DisplayName("should resolve SessionStatus from request attribute")
    void shouldResolveFromRequestAttribute() throws Exception {
        Method method = getClass().getDeclaredMethod("withSessionStatus", SessionStatus.class);
        MethodParameter param = new MethodParameter(method, 0);

        HttpServletRequest request = mock(HttpServletRequest.class);
        SimpleSessionStatus status = new SimpleSessionStatus();
        when(request.getAttribute(SessionStatusArgumentResolver.SESSION_STATUS_ATTRIBUTE))
                .thenReturn(status);

        Object result = resolver.resolveArgument(param, request);

        assertThat(result).isSameAs(status);
    }

    @Test
    @DisplayName("should throw when SessionStatus not found in request")
    void shouldThrow_WhenSessionStatusNotFound() throws Exception {
        Method method = getClass().getDeclaredMethod("withSessionStatus", SessionStatus.class);
        MethodParameter param = new MethodParameter(method, 0);

        HttpServletRequest request = mock(HttpServletRequest.class);
        when(request.getAttribute(SessionStatusArgumentResolver.SESSION_STATUS_ATTRIBUTE))
                .thenReturn(null);

        assertThatThrownBy(() -> resolver.resolveArgument(param, request))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("SessionStatus");
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/RedirectAttributesArgumentResolverTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.flash.SimpleRedirectAttributes;
import com.simplespringmvc.mapping.MethodParameter;
import jakarta.servlet.http.HttpServletRequest;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

/**
 * Unit tests for {@link RedirectAttributesArgumentResolver}.
 */
class RedirectAttributesArgumentResolverTest {

    private final RedirectAttributesArgumentResolver resolver = new RedirectAttributesArgumentResolver();

    // ─── Test methods ────────────────────────────────────────────────

    @SuppressWarnings("unused")
    static void withRedirectAttrs(SimpleRedirectAttributes attrs) {}

    @SuppressWarnings("unused")
    static void withString(String plain) {}

    // ─── Tests ───────────────────────────────────────────────────────

    @Test
    @DisplayName("should support SimpleRedirectAttributes parameter")
    void shouldSupportRedirectAttributes() throws Exception {
        Method method = getClass().getDeclaredMethod("withRedirectAttrs", SimpleRedirectAttributes.class);
        MethodParameter param = new MethodParameter(method, 0);

        assertThat(resolver.supportsParameter(param)).isTrue();
    }

    @Test
    @DisplayName("should not support non-SimpleRedirectAttributes parameter")
    void shouldNotSupportOtherTypes() throws Exception {
        Method method = getClass().getDeclaredMethod("withString", String.class);
        MethodParameter param = new MethodParameter(method, 0);

        assertThat(resolver.supportsParameter(param)).isFalse();
    }

    @Test
    @DisplayName("should create and store redirect attributes in request")
    void shouldCreateAndStoreInRequest() throws Exception {
        Method method = getClass().getDeclaredMethod("withRedirectAttrs", SimpleRedirectAttributes.class);
        MethodParameter param = new MethodParameter(method, 0);

        HttpServletRequest request = mock(HttpServletRequest.class);

        Object result = resolver.resolveArgument(param, request);

        assertThat(result).isInstanceOf(SimpleRedirectAttributes.class);
        verify(request).setAttribute(
                eq(RedirectAttributesArgumentResolver.REDIRECT_ATTRIBUTES_ATTRIBUTE),
                same(result));
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/SessionManagementIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.*;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.flash.FlashMap;
import com.simplespringmvc.flash.SessionFlashMapManager;
import com.simplespringmvc.flash.SimpleRedirectAttributes;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import com.simplespringmvc.session.SessionStatus;
import com.simplespringmvc.view.ModelAndView;
import jakarta.servlet.Servlet;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.mockito.invocation.InvocationOnMock;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

/**
 * Integration tests for Session Management & Flash Attributes (ch18).
 *
 * Tests the full pipeline: DispatcherServlet → HandlerAdapter → SessionAttributes
 * lifecycle, and the flash map lifecycle across redirect requests.
 */
class SessionManagementIntegrationTest {

    // ─── Test POJOs ──────────────────────────────────────────────────

    public static class UserForm {
        private String name;
        private int age;

        public UserForm() {}

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
    }

    // ─── Test controllers ────────────────────────────────────────────

    @Controller
    @SessionAttributes("userForm")
    static class WizardController {

        @RequestMapping(path = "/wizard/step1", method = "POST")
        public ModelAndView step1(@ModelAttribute("userForm") UserForm form) {
            return new ModelAndView("step1")
                    .addObject("userForm", form);
        }

        @RequestMapping(path = "/wizard/step2", method = "POST")
        public ModelAndView step2(@ModelAttribute("userForm") UserForm form) {
            return new ModelAndView("step2")
                    .addObject("userForm", form);
        }

        @RequestMapping(path = "/wizard/finish", method = "POST")
        public String finish(SessionStatus status) {
            status.setComplete();
            return "redirect:/wizard/done";
        }
    }

    @Controller
    static class RedirectController {

        @RequestMapping(path = "/save", method = "POST")
        public String save(SimpleRedirectAttributes redirectAttrs) {
            redirectAttrs.addFlashAttribute("message", "Saved successfully!");
            redirectAttrs.addFlashAttribute("count", 42);
            return "redirect:/list";
        }

        @RequestMapping(path = "/list", method = "GET")
        public String list() {
            return "list-view";
        }
    }

    @Controller
    static class SessionAttrController {

        @RequestMapping(path = "/profile", method = "GET")
        public String profile(@SessionAttribute("currentUser") String user) {
            return "profile: " + user;
        }

        @RequestMapping(path = "/optional", method = "GET")
        public String optional(
                @SessionAttribute(value = "optional", required = false) String val) {
            return "value: " + val;
        }
    }

    // ─── Test setup ──────────────────────────────────────────────────

    private SimpleBeanContainer beanContainer;
    private SimpleDispatcherServlet servlet;
    private HttpSession session;

    @BeforeEach
    void setUp() throws ServletException {
        beanContainer = new SimpleBeanContainer();
        beanContainer.registerBean(new WizardController());
        beanContainer.registerBean(new RedirectController());
        beanContainer.registerBean(new SessionAttrController());

        servlet = new SimpleDispatcherServlet(beanContainer);
        servlet.setFlashMapManager(new SessionFlashMapManager());
        servlet.init();

        session = mock(HttpSession.class);
    }

    /**
     * Create a mock HttpServletRequest that tracks setAttribute/getAttribute
     * calls using a HashMap. This is needed because the DispatcherServlet and
     * HandlerAdapter communicate through request attributes (e.g., session
     * attributes map, SessionStatus, FlashMap).
     */
    private HttpServletRequest mockRequest(String method, String uri) {
        HttpServletRequest request = mock(HttpServletRequest.class);
        when(request.getMethod()).thenReturn(method);
        when(request.getRequestURI()).thenReturn(uri);
        when(request.getSession()).thenReturn(session);
        when(request.getSession(false)).thenReturn(session);

        // Track request attributes in a HashMap
        Map<String, Object> attributes = new HashMap<>();
        doAnswer(invocation -> {
            String name = invocation.getArgument(0);
            Object value = invocation.getArgument(1);
            attributes.put(name, value);
            return null;
        }).when(request).setAttribute(anyString(), any());

        when(request.getAttribute(anyString())).thenAnswer(invocation -> {
            String name = invocation.getArgument(0);
            return attributes.get(name);
        });

        return request;
    }

    private HttpServletResponse mockResponse() throws Exception {
        HttpServletResponse response = mock(HttpServletResponse.class);
        StringWriter writer = new StringWriter();
        when(response.getWriter()).thenReturn(new PrintWriter(writer));
        return response;
    }

    private HttpServletResponse mockResponseWithWriter(StringWriter writer) throws Exception {
        HttpServletResponse response = mock(HttpServletResponse.class);
        when(response.getWriter()).thenReturn(new PrintWriter(writer));
        return response;
    }

    // ─── Session Attributes Tests ────────────────────────────────────

    @Nested
    @DisplayName("@SessionAttributes lifecycle")
    class SessionAttributesLifecycle {

        @Test
        @DisplayName("should store model attribute in session after handler invocation")
        void shouldStoreModelAttributeInSession() throws Exception {
            HttpServletRequest request = mockRequest("POST", "/wizard/step1");
            when(request.getParameterNames()).thenReturn(
                    java.util.Collections.enumeration(java.util.List.of("name")));
            when(request.getParameter("name")).thenReturn("Alice");

            HttpServletResponse response = mockResponse();

            ((Servlet) servlet).service(request, response);

            // Session should have userForm stored
            verify(session).setAttribute(eq("userForm"), argThat(arg ->
                    arg instanceof UserForm form && "Alice".equals(form.getName())));
        }

        @Test
        @DisplayName("should retrieve model attribute from session on next request")
        void shouldRetrieveFromSession() throws Exception {
            // Simulate step2 where userForm is already in session
            UserForm existingForm = new UserForm();
            existingForm.setName("Alice");

            when(session.getAttribute("userForm")).thenReturn(existingForm);

            HttpServletRequest request = mockRequest("POST", "/wizard/step2");
            when(request.getParameterNames()).thenReturn(
                    java.util.Collections.enumeration(java.util.List.of("age")));
            when(request.getParameter("age")).thenReturn("30");

            HttpServletResponse response = mockResponse();

            ((Servlet) servlet).service(request, response);

            // The same object should be updated with age, keeping name
            assertThat(existingForm.getName()).isEqualTo("Alice");
            assertThat(existingForm.getAge()).isEqualTo(30);
        }

        @Test
        @DisplayName("should cleanup session attributes on SessionStatus.setComplete()")
        void shouldCleanupOnComplete() throws Exception {
            HttpServletRequest request = mockRequest("POST", "/wizard/finish");
            when(request.getParameterNames()).thenReturn(
                    java.util.Collections.emptyEnumeration());
            HttpServletResponse response = mockResponse();

            ((Servlet) servlet).service(request, response);

            // Should remove userForm from session
            verify(session).removeAttribute("userForm");
        }
    }

    // ─── @SessionAttribute Tests ─────────────────────────────────────

    @Nested
    @DisplayName("@SessionAttribute parameter injection")
    class SessionAttributeInjection {

        @Test
        @DisplayName("should inject session attribute value into parameter")
        void shouldInjectSessionAttribute() throws Exception {
            when(session.getAttribute("currentUser")).thenReturn("Bob");

            HttpServletRequest request = mockRequest("GET", "/profile");
            when(request.getParameterNames()).thenReturn(
                    java.util.Collections.emptyEnumeration());

            StringWriter writer = new StringWriter();
            HttpServletResponse response = mockResponseWithWriter(writer);

            ((Servlet) servlet).service(request, response);

            assertThat(writer.toString()).isEqualTo("profile: Bob");
        }

        @Test
        @DisplayName("should inject null for optional missing attribute")
        void shouldInjectNull_WhenOptionalMissing() throws Exception {
            when(session.getAttribute("optional")).thenReturn(null);

            HttpServletRequest request = mockRequest("GET", "/optional");
            when(request.getParameterNames()).thenReturn(
                    java.util.Collections.emptyEnumeration());

            StringWriter writer = new StringWriter();
            HttpServletResponse response = mockResponseWithWriter(writer);

            ((Servlet) servlet).service(request, response);

            assertThat(writer.toString()).isEqualTo("value: null");
        }
    }

    // ─── Flash Attributes Tests ──────────────────────────────────────

    @Nested
    @DisplayName("Flash attributes and redirect")
    class FlashAttributesAndRedirect {

        @Test
        @DisplayName("should redirect and save flash attributes")
        void shouldRedirectAndSaveFlashAttributes() throws Exception {
            HttpServletRequest request = mockRequest("POST", "/save");
            when(request.getParameterNames()).thenReturn(
                    java.util.Collections.emptyEnumeration());
            HttpServletResponse response = mockResponse();

            ((Servlet) servlet).service(request, response);

            // Should redirect
            verify(response).sendRedirect("/list");

            // Should store flash maps in session
            verify(session).setAttribute(
                    eq(SessionFlashMapManager.class.getName() + ".FLASH_MAPS"),
                    argThat(arg -> {
                        @SuppressWarnings("unchecked")
                        var maps = (java.util.List<FlashMap>) arg;
                        if (maps.size() != 1) return false;
                        FlashMap map = maps.get(0);
                        return "Saved successfully!".equals(map.get("message"))
                                && Integer.valueOf(42).equals(map.get("count"))
                                && "/list".equals(map.getTargetRequestPath());
                    }));
        }

        @Test
        @DisplayName("should retrieve flash attributes on redirect target request")
        void shouldRetrieveFlashAttributesOnTarget() throws Exception {
            // Set up flash maps in session (simulating a previous redirect)
            FlashMap flashMap = new FlashMap();
            flashMap.put("message", "Saved successfully!");
            flashMap.setTargetRequestPath("/list");
            flashMap.startExpirationPeriod(180);
            java.util.List<FlashMap> flashMaps = new java.util.ArrayList<>();
            flashMaps.add(flashMap);

            when(session.getAttribute(
                    SessionFlashMapManager.class.getName() + ".FLASH_MAPS"))
                    .thenReturn(flashMaps);

            HttpServletRequest request = mockRequest("GET", "/list");
            when(request.getParameterNames()).thenReturn(
                    java.util.Collections.emptyEnumeration());

            StringWriter writer = new StringWriter();
            HttpServletResponse response = mockResponseWithWriter(writer);

            ((Servlet) servlet).service(request, response);

            // The flash map should be consumed (removed from session)
            verify(session).removeAttribute(
                    SessionFlashMapManager.class.getName() + ".FLASH_MAPS");

            // Flash attributes should be available as request attributes
            verify(request).setAttribute(
                    eq(SimpleDispatcherServlet.INPUT_FLASH_MAP_ATTRIBUTE),
                    argThat(arg -> {
                        @SuppressWarnings("unchecked")
                        Map<String, Object> map = (Map<String, Object>) arg;
                        return "Saved successfully!".equals(map.get("message"));
                    }));
        }

        @Test
        @DisplayName("should handle redirect without flash attributes")
        void shouldHandleRedirectWithoutFlashAttributes() throws Exception {
            // Controller that returns redirect but doesn't use SimpleRedirectAttributes
            @Controller
            class SimpleRedirectController {
                @RequestMapping(path = "/go", method = "GET")
                public String go() {
                    return "redirect:/target";
                }
            }

            SimpleBeanContainer container = new SimpleBeanContainer();
            container.registerBean(new SimpleRedirectController());
            SimpleDispatcherServlet ds = new SimpleDispatcherServlet(container);
            ds.setFlashMapManager(new SessionFlashMapManager());
            ds.init();

            HttpServletRequest request = mockRequest("GET", "/go");
            when(request.getParameterNames()).thenReturn(
                    java.util.Collections.emptyEnumeration());
            HttpServletResponse response = mockResponse();

            ((Servlet) ds).service(request, response);

            verify(response).sendRedirect("/target");
            // No flash map should be saved (output flash map is empty)
        }
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **`@SessionAttributes`** | Type-level annotation declaring which model attributes are transparently stored in the HTTP session between requests |
| **`@SessionAttribute`** | Parameter-level annotation for injecting permanent session attributes (auth tokens, user objects) -- distinct from `@SessionAttributes` |
| **`SessionAttributesHandler`** | Per-controller facade that extracts `@SessionAttributes` metadata, matches attributes by name or type, and manages the retrieve/store/cleanup lifecycle |
| **`SessionStatus.setComplete()`** | Signal from handler method to clean up all managed session attributes, ending the conversational session |
| **`FlashMap`** | A `HashMap` with target path and expiration -- carries flash attributes across exactly one redirect |
| **`SessionFlashMapManager`** | Stores `List<FlashMap>` in the HTTP session, matches by target path, cleans up expired maps |
| **`SimpleRedirectAttributes`** | Controller-facing API for adding flash attributes that the dispatcher copies into the output FlashMap during redirect processing |
| **`redirect:` prefix** | Convention in the view name that triggers HTTP 302 redirect instead of view resolution -- the dispatcher switches code paths based on this prefix |
| **Request attributes as data bus** | Components communicate through request attributes (`SESSION_ATTRIBUTES_KEY`, `SESSION_STATUS_ATTRIBUTE`, `REDIRECT_ATTRIBUTES_ATTRIBUTE`) instead of passing parameters through the call chain |

**Next: Chapter 19 -- CORS Configuration** -- Support Cross-Origin Resource Sharing by intercepting preflight OPTIONS requests and adding CORS headers to responses
