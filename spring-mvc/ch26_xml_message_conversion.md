# Chapter 26: XML Message Conversion

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| The framework has JSON (`JacksonMessageConverter`) and plain text (`PlainTextMessageConverter`) converters. Content negotiation selects between them based on the `Accept` header. | Clients requesting `Accept: application/xml` get a 406 Not Acceptable. There's no way to produce or consume XML — a format still widely used in enterprise integrations, SOAP services, and legacy systems. | Add two XML converters — JAXB (for `@XmlRootElement`-annotated classes) and Jackson XML (for any POJO) — that plug into the existing converter chain with zero changes to the dispatch pipeline. |

---

## 26.1 The Integration Point

The integration point is `SimpleHandlerAdapter`'s constructor, where the converter chain is assembled. This is the ONLY place that changes in the existing codebase — we add two lines to register the new converters.

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`
**Change:** Register `Jaxb2XmlMessageConverter` and `JacksonXmlMessageConverter` after the existing JSON and plain text converters.

```java
// ch14: Register converters in priority order.
// ORDER MATTERS: JacksonMessageConverter first means JSON is the default
// when Accept is */* (most common case). PlainTextMessageConverter handles
// text/plain requests for String return values.
messageConverters.add(new JacksonMessageConverter());
messageConverters.add(new PlainTextMessageConverter());
// ch26: XML converters — JAXB first (for @XmlRootElement classes), then
// Jackson XML (for any POJO). JAXB is more specific (annotation-gated),
// so it should be checked first when both could handle a class.
messageConverters.add(new Jaxb2XmlMessageConverter());
messageConverters.add(new JacksonXmlMessageConverter());
```

Two key decisions here:

1. **JAXB before Jackson XML** — `Jaxb2XmlMessageConverter` is more specific: it only handles `@XmlRootElement`-annotated classes. `JacksonXmlMessageConverter` handles *any* class. By putting JAXB first, classes with JAXB annotations get the standards-based serialization, while plain POJOs fall through to Jackson XML. This mirrors the real framework's converter ordering.

2. **XML after JSON and text** — JSON remains the default for `*/*` requests because `JacksonMessageConverter` is first in the list. XML converters only activate when the client explicitly requests `application/xml` or `text/xml`.

This **connects the XML converters to the existing dispatch pipeline** (`ResponseBodyReturnValueHandler`, `RequestBodyArgumentResolver`, `ContentNegotiationManager`). None of those components need modification — they already iterate converters and match by media type. To make this integration point work, we need to build:

- **`Jaxb2XmlMessageConverter`** — JAXB-based converter for `@XmlRootElement` classes
- **`JacksonXmlMessageConverter`** — Jackson XML converter for any POJO
- **`MediaType.TEXT_XML`** — the `text/xml` media type constant

## 26.2 MediaType Enhancement

Before building the converters, we need a `TEXT_XML` constant. XML has two standard media types: `application/xml` (the canonical form) and `text/xml` (the legacy form — still used by many clients and SOAP services).

**Modifying:** `src/main/java/com/simplespringmvc/http/MediaType.java`
**Change:** Add `TEXT_XML` constant after the existing `APPLICATION_XML`.

```java
/** XML: {@code application/xml} */
public static final MediaType APPLICATION_XML = new MediaType("application", "xml");

/** XML: {@code text/xml} — ch26: alternative XML media type used by some clients */
public static final MediaType TEXT_XML = new MediaType("text", "xml");
```

## 26.3 Jaxb2XmlMessageConverter — Standards-Based XML

The JAXB converter handles classes explicitly annotated with `@XmlRootElement`. This is the standards-based approach (Jakarta XML Binding) — ideal for schema-first development and WSDL contracts.

**New file:** `src/main/java/com/simplespringmvc/converter/Jaxb2XmlMessageConverter.java`

```java
package com.simplespringmvc.converter;

import com.simplespringmvc.http.MediaType;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.xml.bind.JAXBContext;
import jakarta.xml.bind.JAXBException;
import jakarta.xml.bind.Marshaller;
import jakarta.xml.bind.Unmarshaller;
import jakarta.xml.bind.annotation.XmlRootElement;

import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class Jaxb2XmlMessageConverter implements HttpMessageConverter {

    private static final List<MediaType> SUPPORTED =
            List.of(MediaType.APPLICATION_XML, MediaType.TEXT_XML);

    // JAXBContext is expensive to create but thread-safe — cache per class
    private final ConcurrentMap<Class<?>, JAXBContext> jaxbContexts = new ConcurrentHashMap<>(64);

    @Override
    public List<MediaType> getSupportedMediaTypes() { return SUPPORTED; }

    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        if (!clazz.isAnnotationPresent(XmlRootElement.class)) return false;
        if (mediaType == null) return true;
        return isXmlCompatible(mediaType);
    }

    @Override
    public Object read(Class<?> clazz, HttpServletRequest request) throws Exception {
        Unmarshaller unmarshaller = createUnmarshaller(clazz);
        return unmarshaller.unmarshal(request.getInputStream());
    }

    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        if (!clazz.isAnnotationPresent(XmlRootElement.class)) return false;
        if (mediaType == null) return true;
        return isXmlCompatible(mediaType);
    }

    @Override
    public void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception {
        String contentType = (mediaType != null) ? mediaType.toString() : MediaType.APPLICATION_XML.toString();
        response.setContentType(contentType);
        response.setCharacterEncoding("UTF-8");

        Marshaller marshaller = createMarshaller(value.getClass());
        marshaller.setProperty(Marshaller.JAXB_ENCODING, "UTF-8");
        marshaller.setProperty(Marshaller.JAXB_FRAGMENT, true);
        marshaller.marshal(value, response.getWriter());
        response.getWriter().flush();
    }

    private Marshaller createMarshaller(Class<?> clazz) throws JAXBException {
        return getJaxbContext(clazz).createMarshaller();
    }

    private Unmarshaller createUnmarshaller(Class<?> clazz) throws JAXBException {
        return getJaxbContext(clazz).createUnmarshaller();
    }

    private JAXBContext getJaxbContext(Class<?> clazz) throws JAXBException {
        JAXBContext context = jaxbContexts.get(clazz);
        if (context != null) return context;
        context = JAXBContext.newInstance(clazz);
        jaxbContexts.putIfAbsent(clazz, context);
        return jaxbContexts.get(clazz);
    }

    private boolean isXmlCompatible(MediaType mediaType) {
        return MediaType.APPLICATION_XML.isCompatibleWith(mediaType)
                || MediaType.TEXT_XML.isCompatibleWith(mediaType);
    }
}
```

The critical design decision is the **caching strategy**: `JAXBContext` instances are expensive to create (they introspect annotations, generate internal mappings) but are thread-safe. So we cache them per class in a `ConcurrentHashMap`. Marshallers/Unmarshallers are cheap to create but NOT thread-safe, so we create fresh ones per request.

## 26.4 JacksonXmlMessageConverter — Annotation-Free XML

The Jackson XML converter works with **any** Java class — no annotations required. It uses Jackson's `XmlMapper`, which is the XML counterpart to `ObjectMapper`. The implementation mirrors `JacksonMessageConverter` almost exactly.

**New file:** `src/main/java/com/simplespringmvc/converter/JacksonXmlMessageConverter.java`

```java
package com.simplespringmvc.converter;

import com.fasterxml.jackson.dataformat.xml.XmlMapper;
import com.simplespringmvc.http.MediaType;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.List;

public class JacksonXmlMessageConverter implements HttpMessageConverter {

    private static final List<MediaType> SUPPORTED =
            List.of(MediaType.APPLICATION_XML, MediaType.TEXT_XML);

    private final XmlMapper xmlMapper;

    public JacksonXmlMessageConverter() {
        this.xmlMapper = new XmlMapper();
    }

    public JacksonXmlMessageConverter(XmlMapper xmlMapper) {
        this.xmlMapper = xmlMapper;
    }

    @Override
    public List<MediaType> getSupportedMediaTypes() { return SUPPORTED; }

    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        if (mediaType == null) return true;
        return isXmlCompatible(mediaType);
    }

    @Override
    public Object read(Class<?> clazz, HttpServletRequest request) throws Exception {
        return xmlMapper.readValue(request.getInputStream(), clazz);
    }

    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        if (mediaType == null) return true;
        return isXmlCompatible(mediaType);
    }

    @Override
    public void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception {
        String contentType = (mediaType != null) ? mediaType.toString() : MediaType.APPLICATION_XML.toString();
        response.setContentType(contentType);
        response.setCharacterEncoding("UTF-8");
        xmlMapper.writeValue(response.getWriter(), value);
    }

    public XmlMapper getXmlMapper() { return xmlMapper; }

    private boolean isXmlCompatible(MediaType mediaType) {
        return MediaType.APPLICATION_XML.isCompatibleWith(mediaType)
                || MediaType.TEXT_XML.isCompatibleWith(mediaType);
    }
}
```

Notice how `canRead`/`canWrite` accept *any* class (no annotation check) — this is the key difference from the JAXB converter. Jackson XML uses getter/setter conventions, just like Jackson JSON.

## 26.5 Build Dependencies

**Modifying:** `build.gradle`
**Change:** Add Jackson XML and JAXB dependencies.

```groovy
// Jackson XML — ch26: XML message conversion via XmlMapper (mirrors JSON converter pattern)
implementation 'com.fasterxml.jackson.dataformat:jackson-dataformat-xml:2.18.3'

// JAXB — ch26: XML message conversion via jakarta.xml.bind for @XmlRootElement classes
implementation 'jakarta.xml.bind:jakarta.xml.bind-api:4.0.2'
runtimeOnly 'org.glassfish.jaxb:jaxb-runtime:4.0.5'
```

Two separate dependency groups:
- **Jackson XML** (`jackson-dataformat-xml`) — a Jackson module that plugs `XmlMapper` into Jackson's serialization engine. Same version as `jackson-databind`.
- **JAXB** — the API (`jakarta.xml.bind-api`) is compile-time, the runtime implementation (`jaxb-runtime`) is `runtimeOnly` because JAXB uses a service loader to discover it.

## 26.6 Try It Yourself

<details>
<summary>Challenge 1: Which converter wins for a @XmlRootElement class with Accept: application/xml?</summary>

The `Jaxb2XmlMessageConverter` wins because:
1. It's registered *before* `JacksonXmlMessageConverter` in the converter list
2. It checks `canWrite(clazz, mediaType)` — the class has `@XmlRootElement` and the media type is XML-compatible
3. The first converter that returns `true` for `canWrite` is used

If you remove the JAXB converter from the chain, `JacksonXmlMessageConverter` would handle it instead — it accepts any class.

</details>

<details>
<summary>Challenge 2: What happens if a client sends Accept: application/xml but the controller returns a String?</summary>

The `Jaxb2XmlMessageConverter` rejects it (`String.class` has no `@XmlRootElement`). The `JacksonXmlMessageConverter` accepts it (any class with XML media type). So Jackson XML serializes the String as XML:

```xml
<String>hello</String>
```

This is technically valid XML, but likely not what the client expects. In practice, String return values from `@ResponseBody` methods are typically handled by `PlainTextMessageConverter` when the client sends `Accept: text/plain`, or by the view resolution pipeline.

</details>

<details>
<summary>Challenge 3: Build a converter for YAML (application/x-yaml) using Jackson's jackson-dataformat-yaml module</summary>

It follows the exact same pattern as `JacksonXmlMessageConverter`:

```java
public class JacksonYamlMessageConverter implements HttpMessageConverter {
    private static final MediaType APPLICATION_YAML = new MediaType("application", "x-yaml");
    private static final List<MediaType> SUPPORTED = List.of(APPLICATION_YAML);
    private final YAMLMapper yamlMapper = new YAMLMapper();

    @Override
    public List<MediaType> getSupportedMediaTypes() { return SUPPORTED; }

    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        return mediaType == null || APPLICATION_YAML.isCompatibleWith(mediaType);
    }

    @Override
    public Object read(Class<?> clazz, HttpServletRequest request) throws Exception {
        return yamlMapper.readValue(request.getInputStream(), clazz);
    }

    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        return mediaType == null || APPLICATION_YAML.isCompatibleWith(mediaType);
    }

    @Override
    public void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception {
        response.setContentType(APPLICATION_YAML.toString());
        response.setCharacterEncoding("UTF-8");
        yamlMapper.writeValue(response.getWriter(), value);
    }
}
```

This is the extensibility that the `HttpMessageConverter` pattern enables — new formats require only a new converter class and one registration line.

</details>

## 26.7 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/converter/Jaxb2XmlMessageConverterTest.java`

Tests the JAXB converter in isolation — annotation gating, media type checks, marshal/unmarshal, and round-trip fidelity.

```java
@Test
void shouldSupportReadForXmlRootElementClass_WhenMediaTypeIsApplicationXml() {
    assertThat(converter.canRead(JaxbProduct.class, MediaType.APPLICATION_XML)).isTrue();
}

@Test
void shouldNotSupportRead_WhenClassLacksXmlRootElement() {
    assertThat(converter.canRead(PlainPojo.class, MediaType.APPLICATION_XML)).isFalse();
}

@Test
void shouldMarshalObjectToXml_WhenWriting() throws Exception {
    JaxbProduct product = new JaxbProduct();
    product.setName("Widget");
    product.setPrice(9.99);

    MockHttpServletResponse response = new MockHttpServletResponse();
    converter.write(product, MediaType.APPLICATION_XML, response);

    assertThat(response.getContentAsString()).contains("<name>Widget</name>");
    assertThat(response.getContentType()).startsWith("application/xml");
}

@Test
void shouldRoundTrip_WhenMarshallingAndUnmarshalling() throws Exception {
    // Marshal → XML string → Unmarshal → assert equal values
}
```

**New file:** `src/test/java/com/simplespringmvc/converter/JacksonXmlMessageConverterTest.java`

Tests the Jackson XML converter — no annotation requirements, POJO serialization, media type gating.

```java
@Test
void shouldSupportReadForAnyClass_WhenMediaTypeIsApplicationXml() {
    assertThat(converter.canRead(Product.class, MediaType.APPLICATION_XML)).isTrue();
}

@Test
void shouldNotSupportRead_WhenMediaTypeIsJson() {
    assertThat(converter.canRead(Product.class, MediaType.APPLICATION_JSON)).isFalse();
}

@Test
void shouldSerializePojoToXml_WhenWriting() throws Exception {
    Product product = new Product("Widget", 9.99);
    MockHttpServletResponse response = new MockHttpServletResponse();
    converter.write(product, MediaType.APPLICATION_XML, response);

    assertThat(response.getContentAsString()).contains("<name>Widget</name>");
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/XmlMessageConversionIntegrationTest.java`

Tests how XML converters integrate with content negotiation and the converter chain.

```java
@Test
void shouldSelectJsonConverter_WhenAcceptHeaderIsApplicationJson() {
    HttpMessageConverter selected = findWriteConverter(Product.class, MediaType.APPLICATION_JSON);
    assertThat(selected).isInstanceOf(JacksonMessageConverter.class);
}

@Test
void shouldSelectJaxbConverter_WhenAcceptIsXmlAndClassHasXmlRootElement() {
    HttpMessageConverter selected = findWriteConverter(JaxbItem.class, MediaType.APPLICATION_XML);
    assertThat(selected).isInstanceOf(Jaxb2XmlMessageConverter.class);
}

@Test
void shouldSelectXmlConverter_WhenAcceptHeaderIsApplicationXml() {
    HttpMessageConverter selected = findWriteConverter(Product.class, MediaType.APPLICATION_XML);
    // Plain POJO → falls through to JacksonXmlMessageConverter
    assertThat(selected).isInstanceOf(JacksonXmlMessageConverter.class);
}

@Test
void shouldRegisterXmlConverters_WhenAdapterIsCreated() {
    SimpleHandlerAdapter adapter = new SimpleHandlerAdapter();
    List<HttpMessageConverter> registered = adapter.getMessageConverters();
    assertThat(registered).hasSize(4);
    assertThat(registered.get(2)).isInstanceOf(Jaxb2XmlMessageConverter.class);
    assertThat(registered.get(3)).isInstanceOf(JacksonXmlMessageConverter.class);
}
```

**Run:** `./gradlew test` — expected: all 910 tests pass (including all prior features' tests)

---

## 26.8 Why This Works

> ★ **Insight** -------------------------------------------
> **Open/Closed Principle in action.** The `HttpMessageConverter` interface is the extension point. The dispatch pipeline (`ResponseBodyReturnValueHandler`, `RequestBodyArgumentResolver`, `ContentNegotiationManager`) is closed for modification — none of those classes changed. Adding XML support is purely additive: new converter classes + registration. This is what makes Spring's converter architecture genuinely extensible. The pattern generalizes to any wire format: YAML, Protobuf, CSV, MessagePack — each is just another `HttpMessageConverter` implementation.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **Two XML converters serve different audiences.** JAXB is annotation-gated (`@XmlRootElement`) — it gives you standards-based XML fidelity, schema validation, and namespace control. Jackson XML is annotation-free — it gives you "JSON but in XML" with zero ceremony. The real framework registers both and lets the `canRead`/`canWrite` gating determine which one handles each class. In the converter chain, the more specific converter (JAXB) comes first, so it gets priority for annotated classes. This is the Template Method pattern applied to converter selection.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> **JAXBContext caching is critical for performance.** `JAXBContext.newInstance()` is the most expensive operation in JAXB — it introspects annotations, builds internal schema mappings, and potentially generates bytecode. A single creation can take 10-50ms. But the resulting context is immutable and thread-safe. The real framework caches contexts in a `ConcurrentHashMap` (capacity 64) keyed by class, giving O(1) amortized access after warmup. Marshallers/Unmarshallers are NOT cached because they hold mutable state (character encoding, formatting options) and are NOT thread-safe — but they're cheap to create from a cached context (~0.1ms).
> -----------------------------------------------------------

## 26.9 What We Enhanced

| Aspect | Before (ch14/ch25) | Current (ch26) | Real Framework |
|--------|-------------------|----------------|----------------|
| **Converter count** | 2 (JSON + PlainText) | 4 (JSON + PlainText + JAXB XML + Jackson XML) | 8+ default converters including XML, form, resource, byte array |
| **XML support** | None — `Accept: application/xml` → 406 | Both JAXB and Jackson XML converters handle `application/xml` and `text/xml` | Full XML support with XXE protection, namespace handling, schema validation |
| **Media types** | `application/json`, `text/plain` | + `application/xml`, `text/xml` | + `application/*+xml`, `text/*+xml` suffix matching |
| **XXE protection** | N/A | Not implemented (simplification) | `Jaxb2RootElementHttpMessageConverter.processSource()` wraps in defensive SAXSource; `JacksonXmlHttpMessageConverter.defensiveXmlFactory()` uses StAX defensive factory |

## 26.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `Jaxb2XmlMessageConverter` | `Jaxb2RootElementHttpMessageConverter` | `Jaxb2RootElementHttpMessageConverter.java:1` | Real version has 3-layer hierarchy (AbstractXml → AbstractJaxb2 → concrete), supports `@XmlType` + `JAXBElement`, has XXE protection via SAXSource |
| `JacksonXmlMessageConverter` | `JacksonXmlHttpMessageConverter` | `JacksonXmlHttpMessageConverter.java:1` | Real version extends `AbstractJacksonHttpMessageConverter<XmlMapper>` (shared with JSON), has defensive XML factory, module auto-discovery, problem detail mixin |
| `jaxbContexts` cache | `AbstractJaxb2HttpMessageConverter.jaxbContexts` | `AbstractJaxb2HttpMessageConverter.java:52` | Same pattern — `ConcurrentMap<Class<?>, JAXBContext>` with capacity 64 |
| `canRead` checks `@XmlRootElement` | `canRead` checks `@XmlRootElement` OR `@XmlType` | `Jaxb2RootElementHttpMessageConverter.java:100` | Real version also supports `@XmlType` (unmarshalled via `JAXBElement` wrapper) |
| `canWrite` checks `@XmlRootElement` | `canWrite` checks `@XmlRootElement` OR `JAXBElement.class` | `Jaxb2RootElementHttpMessageConverter.java:115` | Real version supports `JAXBElement` for wrapping `@XmlType`-only classes |
| `MediaType.TEXT_XML` | `MediaType.TEXT_XML` | `MediaType.java:189` | Same constant |

## 26.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/converter/Jaxb2XmlMessageConverter.java` [NEW]

```java
package com.simplespringmvc.converter;

import com.simplespringmvc.http.MediaType;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.xml.bind.JAXBContext;
import jakarta.xml.bind.JAXBException;
import jakarta.xml.bind.Marshaller;
import jakarta.xml.bind.Unmarshaller;
import jakarta.xml.bind.annotation.XmlRootElement;

import java.io.StringWriter;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

/**
 * An {@link HttpMessageConverter} that uses JAXB ({@code jakarta.xml.bind}) to
 * marshal and unmarshal objects annotated with {@link XmlRootElement}.
 *
 * Maps to: {@code org.springframework.http.converter.xml.Jaxb2RootElementHttpMessageConverter}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   AbstractHttpMessageConverter
 *     └── AbstractXmlHttpMessageConverter       (wraps IO in Source/Result)
 *           └── AbstractJaxb2HttpMessageConverter (caches JAXBContext, creates Marshaller/Unmarshaller)
 *                 └── Jaxb2RootElementHttpMessageConverter (canRead checks @XmlRootElement/@XmlType)
 * </pre>
 *
 * <h3>Key design decisions in the real framework:</h3>
 * <ul>
 *   <li>{@code JAXBContext} instances are cached per class — they are expensive to create
 *       but thread-safe, so a {@link ConcurrentHashMap} provides lock-free caching</li>
 *   <li>{@code Marshaller}/{@code Unmarshaller} are NOT cached — they are cheap to create
 *       and NOT thread-safe</li>
 *   <li>{@code canRead()} checks for {@code @XmlRootElement} or {@code @XmlType} —
 *       JAXB can unmarshal both, wrapping @XmlType in a JAXBElement</li>
 *   <li>{@code canWrite()} checks for {@code @XmlRootElement} only — JAXB cannot
 *       marshal @XmlType without a root element wrapper</li>
 *   <li>XXE protection: the real framework wraps input in a SAXSource with DTD
 *       and external entity processing disabled</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>Only supports classes annotated with {@code @XmlRootElement} (no @XmlType, no JAXBElement)</li>
 *   <li>No XXE protection (the real framework wraps XML input in a defensive SAXSource)</li>
 *   <li>No customizeMarshaller/customizeUnmarshaller hooks</li>
 *   <li>Reads/writes directly from/to streams (no Source/Result abstraction)</li>
 * </ul>
 */
public class Jaxb2XmlMessageConverter implements HttpMessageConverter {

    private static final List<MediaType> SUPPORTED =
            List.of(MediaType.APPLICATION_XML, MediaType.TEXT_XML);

    /**
     * Cache of JAXBContext instances per class.
     *
     * Maps to: {@code AbstractJaxb2HttpMessageConverter.jaxbContexts}
     *
     * JAXBContext.newInstance() is expensive (introspects annotations, generates
     * internal mappings), but the resulting context is thread-safe and reusable.
     * ConcurrentHashMap.computeIfAbsent() provides lock-free reads after first creation.
     */
    private final ConcurrentMap<Class<?>, JAXBContext> jaxbContexts = new ConcurrentHashMap<>(64);

    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return SUPPORTED;
    }

    /**
     * Can read classes annotated with {@code @XmlRootElement} when the content type
     * is XML-compatible (or unknown).
     *
     * Maps to: {@code Jaxb2RootElementHttpMessageConverter.canRead(Class, MediaType)}
     */
    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        if (!clazz.isAnnotationPresent(XmlRootElement.class)) {
            return false;
        }
        if (mediaType == null) {
            return true;
        }
        return isXmlCompatible(mediaType);
    }

    /**
     * Deserialize XML from the request body into an object of the given type.
     *
     * Maps to: {@code Jaxb2RootElementHttpMessageConverter.readFromSource()}
     */
    @Override
    public Object read(Class<?> clazz, HttpServletRequest request) throws Exception {
        Unmarshaller unmarshaller = createUnmarshaller(clazz);
        return unmarshaller.unmarshal(request.getInputStream());
    }

    /**
     * Can write classes annotated with {@code @XmlRootElement} when the target
     * media type is XML-compatible (or unknown).
     *
     * Maps to: {@code Jaxb2RootElementHttpMessageConverter.canWrite(Class, MediaType)}
     *
     * Note: the real framework also supports JAXBElement wrappers for writing
     * @XmlType classes — we skip that for simplicity.
     */
    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        if (!clazz.isAnnotationPresent(XmlRootElement.class)) {
            return false;
        }
        if (mediaType == null) {
            return true;
        }
        return isXmlCompatible(mediaType);
    }

    /**
     * Serialize the given object to XML and write to the response.
     *
     * Maps to: {@code Jaxb2RootElementHttpMessageConverter.writeToResult()}
     */
    @Override
    public void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception {
        String contentType = (mediaType != null) ? mediaType.toString() : MediaType.APPLICATION_XML.toString();
        response.setContentType(contentType);
        response.setCharacterEncoding("UTF-8");

        Marshaller marshaller = createMarshaller(value.getClass());
        marshaller.setProperty(Marshaller.JAXB_ENCODING, "UTF-8");
        // Omit XML declaration — the Content-Type header already declares encoding
        marshaller.setProperty(Marshaller.JAXB_FRAGMENT, true);
        marshaller.marshal(value, response.getWriter());
        response.getWriter().flush();
    }

    /**
     * Create a JAXB Marshaller for the given class.
     * Marshallers are NOT cached — they are cheap to create and NOT thread-safe.
     *
     * Maps to: {@code AbstractJaxb2HttpMessageConverter.createMarshaller()}
     */
    private Marshaller createMarshaller(Class<?> clazz) throws JAXBException {
        return getJaxbContext(clazz).createMarshaller();
    }

    /**
     * Create a JAXB Unmarshaller for the given class.
     *
     * Maps to: {@code AbstractJaxb2HttpMessageConverter.createUnmarshaller()}
     */
    private Unmarshaller createUnmarshaller(Class<?> clazz) throws JAXBException {
        return getJaxbContext(clazz).createUnmarshaller();
    }

    /**
     * Get or create a JAXBContext for the given class.
     * Cached in a ConcurrentHashMap for thread-safe, lock-free access after first creation.
     *
     * Maps to: {@code AbstractJaxb2HttpMessageConverter.getJaxbContext()}
     */
    private JAXBContext getJaxbContext(Class<?> clazz) throws JAXBException {
        JAXBContext context = jaxbContexts.get(clazz);
        if (context != null) {
            return context;
        }
        context = JAXBContext.newInstance(clazz);
        jaxbContexts.putIfAbsent(clazz, context);
        return jaxbContexts.get(clazz);
    }

    /**
     * Check if a media type is XML-compatible: application/xml, text/xml.
     */
    private boolean isXmlCompatible(MediaType mediaType) {
        return MediaType.APPLICATION_XML.isCompatibleWith(mediaType)
                || MediaType.TEXT_XML.isCompatibleWith(mediaType);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/converter/JacksonXmlMessageConverter.java` [NEW]

```java
package com.simplespringmvc.converter;

import com.fasterxml.jackson.dataformat.xml.XmlMapper;
import com.simplespringmvc.http.MediaType;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.util.List;

/**
 * An {@link HttpMessageConverter} that converts between Java objects and XML
 * using Jackson's {@link XmlMapper} (from the {@code jackson-dataformat-xml} module).
 *
 * Maps to: {@code org.springframework.http.converter.xml.JacksonXmlHttpMessageConverter}
 * (Spring 7.0+) and its deprecated predecessor
 * {@code MappingJackson2XmlHttpMessageConverter}
 *
 * <h3>Why two XML converters?</h3>
 * JAXB and Jackson XML serve different use cases:
 * <ul>
 *   <li><strong>JAXB ({@link Jaxb2XmlMessageConverter})</strong> — requires explicit
 *       {@code @XmlRootElement} annotation. Standards-based (Jakarta XML Binding).
 *       Better for schema-first development, WSDL contracts, and Java-to-XML fidelity.</li>
 *   <li><strong>Jackson XML (this class)</strong> — works with ANY Java class (POJOs,
 *       records, collections). Zero annotations required. Better for REST APIs where
 *       you just want "the same JSON shape but in XML".</li>
 * </ul>
 *
 * The real framework registers BOTH converters, and content negotiation picks the
 * right one based on class type (JAXB-annotated vs plain POJO) and requested media type.
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   AbstractHttpMessageConverter
 *     └── AbstractJacksonHttpMessageConverter&lt;XmlMapper&gt;
 *           └── JacksonXmlHttpMessageConverter
 * </pre>
 *
 * The Jackson XML converter mirrors the JSON converter exactly — same canRead/canWrite
 * semantics, same serialization flow — but uses XmlMapper instead of ObjectMapper.
 * This is the power of Jackson's pluggable format modules.
 *
 * Simplifications:
 * <ul>
 *   <li>No defensive XML factory (XXE protection) — the real framework configures
 *       StaxUtils.createDefensiveInputFactory()</li>
 *   <li>No module auto-discovery — uses XmlMapper defaults</li>
 *   <li>No problem detail mixin (RFC 9457)</li>
 * </ul>
 */
public class JacksonXmlMessageConverter implements HttpMessageConverter {

    private static final List<MediaType> SUPPORTED =
            List.of(MediaType.APPLICATION_XML, MediaType.TEXT_XML);

    private final XmlMapper xmlMapper;

    public JacksonXmlMessageConverter() {
        this.xmlMapper = new XmlMapper();
    }

    public JacksonXmlMessageConverter(XmlMapper xmlMapper) {
        this.xmlMapper = xmlMapper;
    }

    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return SUPPORTED;
    }

    /**
     * Jackson XML can deserialize XML into most Java types.
     * When mediaType is null (discovery mode), returns true.
     * When a specific mediaType is given, checks XML compatibility.
     *
     * Maps to: {@code AbstractJacksonHttpMessageConverter.canRead()}
     */
    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        if (mediaType == null) {
            return true;
        }
        return isXmlCompatible(mediaType);
    }

    /**
     * Deserialize XML from the request body into an object of the given type.
     *
     * Maps to: {@code AbstractJacksonHttpMessageConverter.readInternal()}
     */
    @Override
    public Object read(Class<?> clazz, HttpServletRequest request) throws Exception {
        return xmlMapper.readValue(request.getInputStream(), clazz);
    }

    /**
     * Jackson XML can write most Java objects as XML.
     * When mediaType is null (discovery mode), returns true.
     * When a specific mediaType is given, checks XML compatibility.
     *
     * Maps to: {@code AbstractJacksonHttpMessageConverter.canWrite()}
     */
    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        if (mediaType == null) {
            return true;
        }
        return isXmlCompatible(mediaType);
    }

    /**
     * Serialize the given object to XML and write to the response.
     *
     * Maps to: {@code AbstractJacksonHttpMessageConverter.writeInternal()}
     */
    @Override
    public void write(Object value, MediaType mediaType, HttpServletResponse response) throws Exception {
        String contentType = (mediaType != null) ? mediaType.toString() : MediaType.APPLICATION_XML.toString();
        response.setContentType(contentType);
        response.setCharacterEncoding("UTF-8");
        xmlMapper.writeValue(response.getWriter(), value);
    }

    public XmlMapper getXmlMapper() {
        return xmlMapper;
    }

    private boolean isXmlCompatible(MediaType mediaType) {
        return MediaType.APPLICATION_XML.isCompatibleWith(mediaType)
                || MediaType.TEXT_XML.isCompatibleWith(mediaType);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/http/MediaType.java` [MODIFIED]

```java
package com.simplespringmvc.http;

import java.util.*;

/**
 * Represents a MIME type as defined in RFC 2046 — the fundamental data type
 * for content negotiation. Parses media type strings like "application/json"
 * or "text/html;q=0.9" and supports matching, compatibility checks, and
 * quality-based sorting.
 *
 * Maps to: {@code org.springframework.http.MediaType} which extends
 * {@code org.springframework.util.MimeType}
 *
 * <h3>How the real framework does it:</h3>
 * <pre>
 *   MimeType           → stores type/subtype/parameters, matching logic
 *     └── MediaType    → adds quality value, parsing of Accept headers,
 *                        sorting by specificity
 * </pre>
 *
 * We collapse both into a single class. Key operations:
 * <ul>
 *   <li>{@link #includes(MediaType)} — asymmetric: does this type include the other?</li>
 *   <li>{@link #isCompatibleWith(MediaType)} — symmetric: are the two types compatible?</li>
 *   <li>{@link #parseMediaTypes(String)} — parse comma-separated Accept header</li>
 *   <li>{@link #sortBySpecificityAndQuality(List)} — sort for content negotiation</li>
 * </ul>
 *
 * Simplifications:
 * <ul>
 *   <li>No suffix matching (e.g., application/*+xml)</li>
 *   <li>No parameter matching beyond quality value</li>
 *   <li>equals/hashCode use type+subtype only (real framework includes parameters)</li>
 *   <li>No caching of parsed media types</li>
 * </ul>
 */
public class MediaType implements Comparable<MediaType> {

    public static final String WILDCARD_TYPE = "*";
    private static final String PARAM_QUALITY = "q";

    // ─── Common media type constants ──────────────────────────────

    /** Matches any media type: {@code *}{@code /*} */
    public static final MediaType ALL = new MediaType("*", "*");

    /** JSON: {@code application/json} */
    public static final MediaType APPLICATION_JSON = new MediaType("application", "json");

    /** XML: {@code application/xml} */
    public static final MediaType APPLICATION_XML = new MediaType("application", "xml");

    /** XML: {@code text/xml} — ch26: alternative XML media type used by some clients */
    public static final MediaType TEXT_XML = new MediaType("text", "xml");

    /** Plain text: {@code text/plain} */
    public static final MediaType TEXT_PLAIN = new MediaType("text", "plain");

    /** HTML: {@code text/html} */
    public static final MediaType TEXT_HTML = new MediaType("text", "html");

    /** Binary fallback: {@code application/octet-stream} */
    public static final MediaType APPLICATION_OCTET_STREAM = new MediaType("application", "octet-stream");

    // ─── Instance fields ──────────────────────────────────────────

    private final String type;
    private final String subtype;
    private final Map<String, String> parameters;

    public MediaType(String type, String subtype) {
        this(type, subtype, Collections.emptyMap());
    }

    public MediaType(String type, String subtype, Map<String, String> parameters) {
        this.type = type.toLowerCase(Locale.ROOT);
        this.subtype = subtype.toLowerCase(Locale.ROOT);
        this.parameters = Collections.unmodifiableMap(new LinkedHashMap<>(parameters));
    }

    public String getType() {
        return type;
    }

    public String getSubtype() {
        return subtype;
    }

    public Map<String, String> getParameters() {
        return parameters;
    }

    // ─── Quality value ────────────────────────────────────────────

    /**
     * Return the quality value as indicated by the {@code q} parameter.
     * Defaults to {@code 1.0} (highest priority) when not specified.
     *
     * Maps to: {@code MediaType.getQualityValue()} (line 370)
     *
     * Quality values range from 0.0 (not acceptable) to 1.0 (most preferred).
     * Example: {@code text/html;q=0.9} means "I accept HTML but prefer other types".
     */
    public double getQualityValue() {
        String q = parameters.get(PARAM_QUALITY);
        return q != null ? Double.parseDouble(q) : 1.0;
    }

    /**
     * Return a copy with the quality value from another media type.
     * Used during content negotiation to transfer the client's preference
     * weight onto a producible type.
     *
     * Maps to: {@code MediaType.copyQualityValue()} (line 386)
     */
    public MediaType copyQualityValue(MediaType other) {
        double q = other.getQualityValue();
        if (q == 1.0 && !other.parameters.containsKey(PARAM_QUALITY)) {
            return this;
        }
        Map<String, String> params = new LinkedHashMap<>(this.parameters);
        params.put(PARAM_QUALITY, String.valueOf(q));
        return new MediaType(this.type, this.subtype, params);
    }

    /**
     * Return a copy without the quality value parameter.
     * Called before setting the Content-Type header — quality is a client
     * preference, not a response property.
     *
     * Maps to: {@code MediaType.removeQualityValue()} (line 400)
     */
    public MediaType removeQualityValue() {
        if (!parameters.containsKey(PARAM_QUALITY)) {
            return this;
        }
        Map<String, String> params = new LinkedHashMap<>(this.parameters);
        params.remove(PARAM_QUALITY);
        return new MediaType(this.type, this.subtype, params);
    }

    // ─── Wildcard checks ──────────────────────────────────────────

    /** True if the primary type is {@code *} (e.g., {@code * / *}). */
    public boolean isWildcardType() {
        return WILDCARD_TYPE.equals(type);
    }

    /** True if the subtype is {@code *} (e.g., {@code text/*}). */
    public boolean isWildcardSubtype() {
        return WILDCARD_TYPE.equals(subtype) || subtype.startsWith("*+");
    }

    /** True if neither type nor subtype is a wildcard. */
    public boolean isConcrete() {
        return !isWildcardType() && !isWildcardSubtype();
    }

    // ─── Matching ─────────────────────────────────────────────────

    /**
     * Whether this media type includes the given media type.
     * Asymmetric: {@code * / *} includes {@code application/json}, but not vice versa.
     *
     * Maps to: {@code MimeType.includes(MimeType)} (line 253)
     *
     * <pre>
     *   * / *         includes everything
     *   text/*        includes text/plain, text/html
     *   text/plain    includes only text/plain
     *   text/plain    does NOT include * / *
     * </pre>
     */
    public boolean includes(MediaType other) {
        if (other == null) {
            return false;
        }
        if (isWildcardType()) {
            return true;
        }
        if (this.type.equals(other.type)) {
            if (isWildcardSubtype()) {
                return true;
            }
            return this.subtype.equals(other.subtype);
        }
        return false;
    }

    /**
     * Whether this media type is compatible with the given media type.
     * Symmetric: either side can have wildcards.
     *
     * Maps to: {@code MimeType.isCompatibleWith(MimeType)} (line 296)
     *
     * <pre>
     *   application/json.isCompatibleWith(application/json)   → true
     *   application/json.isCompatibleWith(* / *)              → true
     *   * / *.isCompatibleWith(application/json)              → true
     *   text/*.isCompatibleWith(text/plain)                   → true
     *   text/plain.isCompatibleWith(text/*)                   → true
     *   application/json.isCompatibleWith(text/plain)         → false
     * </pre>
     */
    public boolean isCompatibleWith(MediaType other) {
        if (other == null) {
            return false;
        }
        if (isWildcardType() || other.isWildcardType()) {
            return true;
        }
        if (this.type.equals(other.type)) {
            if (isWildcardSubtype() || other.isWildcardSubtype()) {
                return true;
            }
            return this.subtype.equals(other.subtype);
        }
        return false;
    }

    // ─── Parsing ──────────────────────────────────────────────────

    /**
     * Parse a single media type string.
     * Examples: "application/json", "text/html;charset=UTF-8", "text/html;q=0.9"
     *
     * Maps to: {@code MediaType.parseMediaType(String)} → {@code MimeTypeUtils.parseMimeTypeInternal()}
     */
    public static MediaType parseMediaType(String value) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("MediaType must not be blank");
        }
        value = value.trim();
        String[] parts = value.split(";");
        String fullType = parts[0].trim();

        int slashIndex = fullType.indexOf('/');
        String type;
        String subtype;
        if (slashIndex < 0) {
            // No slash — treat as type with wildcard subtype
            type = fullType;
            subtype = "*";
        } else {
            type = fullType.substring(0, slashIndex).trim();
            subtype = fullType.substring(slashIndex + 1).trim();
        }

        Map<String, String> parameters = new LinkedHashMap<>();
        for (int i = 1; i < parts.length; i++) {
            String param = parts[i].trim();
            int eqIdx = param.indexOf('=');
            if (eqIdx > 0) {
                String key = param.substring(0, eqIdx).trim().toLowerCase(Locale.ROOT);
                String val = param.substring(eqIdx + 1).trim();
                // Strip quotes from parameter values
                if (val.length() >= 2 && val.startsWith("\"") && val.endsWith("\"")) {
                    val = val.substring(1, val.length() - 1);
                }
                parameters.put(key, val);
            }
        }

        return new MediaType(type, subtype, parameters);
    }

    /**
     * Parse a comma-separated list of media types (the format used in the Accept header).
     * Example: "text/html, application/json;q=0.9, *&#47;*;q=0.1"
     *
     * Maps to: {@code MediaType.parseMediaTypes(String)} → {@code MimeTypeUtils.tokenize()}
     */
    public static List<MediaType> parseMediaTypes(String value) {
        if (value == null || value.isBlank()) {
            return new ArrayList<>(List.of(ALL));
        }
        List<MediaType> result = new ArrayList<>();
        for (String token : tokenize(value)) {
            result.add(parseMediaType(token));
        }
        return result;
    }

    /**
     * Split a comma-separated media type string.
     * Handles the fact that parameters contain semicolons (not commas),
     * so splitting on comma is safe.
     */
    private static List<String> tokenize(String value) {
        List<String> tokens = new ArrayList<>();
        StringBuilder current = new StringBuilder();
        for (int i = 0; i < value.length(); i++) {
            char c = value.charAt(i);
            if (c == ',') {
                String token = current.toString().trim();
                if (!token.isEmpty()) {
                    tokens.add(token);
                }
                current = new StringBuilder();
            } else {
                current.append(c);
            }
        }
        String last = current.toString().trim();
        if (!last.isEmpty()) {
            tokens.add(last);
        }
        return tokens;
    }

    // ─── Sorting ──────────────────────────────────────────────────

    /**
     * Sort media types by specificity and quality for content negotiation.
     * Most preferred types come first.
     *
     * Sort order:
     * <ol>
     *   <li>Higher quality value first (q=1.0 before q=0.9)</li>
     *   <li>More specific type first (concrete > wildcard subtype > wildcard type)</li>
     *   <li>More parameters = more specific</li>
     * </ol>
     *
     * Maps to: {@code MimeTypeUtils.sortBySpecificity()} combined with
     * quality value comparison from {@code MediaType.isMoreSpecific()}
     */
    public static void sortBySpecificityAndQuality(List<MediaType> mediaTypes) {
        mediaTypes.sort((a, b) -> {
            // Higher quality first
            int qCompare = Double.compare(b.getQualityValue(), a.getQualityValue());
            if (qCompare != 0) return qCompare;

            // More specific first: concrete (2) > wildcard subtype (1) > wildcard type (0)
            int specA = a.isWildcardType() ? 0 : (a.isWildcardSubtype() ? 1 : 2);
            int specB = b.isWildcardType() ? 0 : (b.isWildcardSubtype() ? 1 : 2);
            int specCompare = Integer.compare(specB, specA);
            if (specCompare != 0) return specCompare;

            // More parameters = more specific
            return Integer.compare(b.getParameters().size(), a.getParameters().size());
        });
    }

    // ─── Object methods ───────────────────────────────────────────

    @Override
    public int compareTo(MediaType other) {
        int specA = isWildcardType() ? 0 : (isWildcardSubtype() ? 1 : 2);
        int specB = other.isWildcardType() ? 0 : (other.isWildcardSubtype() ? 1 : 2);
        int specCompare = Integer.compare(specB, specA);
        if (specCompare != 0) return specCompare;
        return Double.compare(other.getQualityValue(), getQualityValue());
    }

    /**
     * Two media types are equal if they have the same type and subtype.
     * Parameters (including quality) are ignored for equality.
     *
     * Simplification: the real framework includes parameters in equals().
     */
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof MediaType that)) return false;
        return type.equals(that.type) && subtype.equals(that.subtype);
    }

    @Override
    public int hashCode() {
        return Objects.hash(type, subtype);
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append(type).append('/').append(subtype);
        parameters.forEach((key, value) -> sb.append(';').append(key).append('=').append(value));
        return sb.toString();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [MODIFIED]

Changes: Added imports for `JacksonXmlMessageConverter` and `Jaxb2XmlMessageConverter`. Added two `messageConverters.add()` calls in both constructors to register the XML converters after JSON and PlainText.

(Full file omitted for brevity — the complete file is in the source tree. The only changes are the import additions and the four `messageConverters.add()` lines shown in section 26.1.)

#### File: `build.gradle` [MODIFIED]

Changes: Added `jackson-dataformat-xml`, `jakarta.xml.bind-api`, and `jaxb-runtime` dependencies.

```groovy
plugins {
    id 'java'
}

group = 'com.simplespringmvc'
version = '1.0-SNAPSHOT'

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

tasks.withType(JavaCompile).configureEach {
    options.compilerArgs.add('-parameters')
}

repositories {
    mavenCentral()
}

dependencies {
    // Embedded Tomcat (Servlet container + WebSocket support)
    implementation 'org.apache.tomcat.embed:tomcat-embed-core:11.0.6'
    implementation 'org.apache.tomcat.embed:tomcat-embed-websocket:11.0.6'

    // Jakarta Servlet API
    compileOnly 'jakarta.servlet:jakarta.servlet-api:6.1.0'

    // Jackson for JSON processing
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.18.3'

    // Jackson XML — ch26: XML message conversion via XmlMapper (mirrors JSON converter pattern)
    implementation 'com.fasterxml.jackson.dataformat:jackson-dataformat-xml:2.18.3'

    // JAXB — ch26: XML message conversion via jakarta.xml.bind for @XmlRootElement classes
    implementation 'jakarta.xml.bind:jakarta.xml.bind-api:4.0.2'
    runtimeOnly 'org.glassfish.jaxb:jaxb-runtime:4.0.5'

    // Jakarta Bean Validation + Hibernate Validator (reference implementation)
    implementation 'org.hibernate.validator:hibernate-validator:8.0.2.Final'
    runtimeOnly 'org.glassfish.expressly:expressly:5.0.0'

    // Testing
    testImplementation 'org.junit.jupiter:junit-jupiter:5.11.4'
    testImplementation 'org.assertj:assertj-core:3.27.3'
    testImplementation 'org.mockito:mockito-core:5.14.2'
    testImplementation 'org.springframework:spring-test:6.2.4'
    testImplementation 'org.springframework:spring-web:6.2.4'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

test {
    useJUnitPlatform()
}
```

### Test Code

#### File: `src/test/java/com/simplespringmvc/converter/Jaxb2XmlMessageConverterTest.java` [NEW]

```java
package com.simplespringmvc.converter;

import com.simplespringmvc.http.MediaType;
import jakarta.xml.bind.annotation.XmlRootElement;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import java.nio.charset.StandardCharsets;

import static org.assertj.core.api.Assertions.assertThat;

class Jaxb2XmlMessageConverterTest {

    private Jaxb2XmlMessageConverter converter;

    @BeforeEach
    void setUp() {
        converter = new Jaxb2XmlMessageConverter();
    }

    @Test
    void shouldSupportReadForXmlRootElementClass_WhenMediaTypeIsNull() {
        assertThat(converter.canRead(JaxbProduct.class, null)).isTrue();
    }

    @Test
    void shouldSupportReadForXmlRootElementClass_WhenMediaTypeIsApplicationXml() {
        assertThat(converter.canRead(JaxbProduct.class, MediaType.APPLICATION_XML)).isTrue();
    }

    @Test
    void shouldSupportReadForXmlRootElementClass_WhenMediaTypeIsTextXml() {
        assertThat(converter.canRead(JaxbProduct.class, MediaType.TEXT_XML)).isTrue();
    }

    @Test
    void shouldNotSupportRead_WhenClassLacksXmlRootElement() {
        assertThat(converter.canRead(PlainPojo.class, MediaType.APPLICATION_XML)).isFalse();
    }

    @Test
    void shouldNotSupportRead_WhenMediaTypeIsJson() {
        assertThat(converter.canRead(JaxbProduct.class, MediaType.APPLICATION_JSON)).isFalse();
    }

    @Test
    void shouldSupportWriteForXmlRootElementClass_WhenMediaTypeIsNull() {
        assertThat(converter.canWrite(JaxbProduct.class, null)).isTrue();
    }

    @Test
    void shouldSupportWriteForXmlRootElementClass_WhenMediaTypeIsApplicationXml() {
        assertThat(converter.canWrite(JaxbProduct.class, MediaType.APPLICATION_XML)).isTrue();
    }

    @Test
    void shouldNotSupportWrite_WhenClassLacksXmlRootElement() {
        assertThat(converter.canWrite(PlainPojo.class, MediaType.APPLICATION_XML)).isFalse();
    }

    @Test
    void shouldNotSupportWrite_WhenMediaTypeIsJson() {
        assertThat(converter.canWrite(JaxbProduct.class, MediaType.APPLICATION_JSON)).isFalse();
    }

    @Test
    void shouldMarshalObjectToXml_WhenWriting() throws Exception {
        JaxbProduct product = new JaxbProduct();
        product.setName("Widget");
        product.setPrice(9.99);

        MockHttpServletResponse response = new MockHttpServletResponse();
        converter.write(product, MediaType.APPLICATION_XML, response);

        String xml = response.getContentAsString();
        assertThat(xml).contains("<jaxbProduct");
        assertThat(xml).contains("<name>Widget</name>");
        assertThat(xml).contains("<price>9.99</price>");
        assertThat(response.getContentType()).startsWith("application/xml");
    }

    @Test
    void shouldSetTextXmlContentType_WhenMediaTypeIsTextXml() throws Exception {
        JaxbProduct product = new JaxbProduct();
        product.setName("Gadget");
        product.setPrice(19.99);

        MockHttpServletResponse response = new MockHttpServletResponse();
        converter.write(product, MediaType.TEXT_XML, response);

        assertThat(response.getContentType()).startsWith("text/xml");
        assertThat(response.getContentAsString()).contains("<name>Gadget</name>");
    }

    @Test
    void shouldUnmarshalXmlToObject_WhenReading() throws Exception {
        String xml = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>"
                + "<jaxbProduct><name>Widget</name><price>9.99</price></jaxbProduct>";

        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setContent(xml.getBytes(StandardCharsets.UTF_8));
        request.setContentType("application/xml");

        Object result = converter.read(JaxbProduct.class, request);

        assertThat(result).isInstanceOf(JaxbProduct.class);
        JaxbProduct product = (JaxbProduct) result;
        assertThat(product.getName()).isEqualTo("Widget");
        assertThat(product.getPrice()).isEqualTo(9.99);
    }

    @Test
    void shouldReturnApplicationXmlAndTextXml_WhenGettingSupportedMediaTypes() {
        assertThat(converter.getSupportedMediaTypes())
                .containsExactly(MediaType.APPLICATION_XML, MediaType.TEXT_XML);
    }

    @Test
    void shouldRoundTrip_WhenMarshallingAndUnmarshalling() throws Exception {
        JaxbProduct original = new JaxbProduct();
        original.setName("Roundtrip Item");
        original.setPrice(42.0);

        MockHttpServletResponse response = new MockHttpServletResponse();
        converter.write(original, MediaType.APPLICATION_XML, response);
        String xml = response.getContentAsString();

        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setContent(xml.getBytes(StandardCharsets.UTF_8));
        request.setContentType("application/xml");

        JaxbProduct deserialized = (JaxbProduct) converter.read(JaxbProduct.class, request);
        assertThat(deserialized.getName()).isEqualTo("Roundtrip Item");
        assertThat(deserialized.getPrice()).isEqualTo(42.0);
    }

    @XmlRootElement
    public static class JaxbProduct {
        private String name;
        private double price;

        public JaxbProduct() {}

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public double getPrice() { return price; }
        public void setPrice(double price) { this.price = price; }
    }

    public static class PlainPojo {
        private String value;
        public String getValue() { return value; }
        public void setValue(String value) { this.value = value; }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/converter/JacksonXmlMessageConverterTest.java` [NEW]

```java
package com.simplespringmvc.converter;

import com.simplespringmvc.http.MediaType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import java.nio.charset.StandardCharsets;

import static org.assertj.core.api.Assertions.assertThat;

class JacksonXmlMessageConverterTest {

    private JacksonXmlMessageConverter converter;

    @BeforeEach
    void setUp() {
        converter = new JacksonXmlMessageConverter();
    }

    @Test
    void shouldSupportReadForAnyClass_WhenMediaTypeIsNull() {
        assertThat(converter.canRead(Product.class, null)).isTrue();
    }

    @Test
    void shouldSupportReadForAnyClass_WhenMediaTypeIsApplicationXml() {
        assertThat(converter.canRead(Product.class, MediaType.APPLICATION_XML)).isTrue();
    }

    @Test
    void shouldSupportReadForAnyClass_WhenMediaTypeIsTextXml() {
        assertThat(converter.canRead(Product.class, MediaType.TEXT_XML)).isTrue();
    }

    @Test
    void shouldNotSupportRead_WhenMediaTypeIsJson() {
        assertThat(converter.canRead(Product.class, MediaType.APPLICATION_JSON)).isFalse();
    }

    @Test
    void shouldNotSupportRead_WhenMediaTypeIsTextPlain() {
        assertThat(converter.canRead(Product.class, MediaType.TEXT_PLAIN)).isFalse();
    }

    @Test
    void shouldSupportWriteForAnyClass_WhenMediaTypeIsNull() {
        assertThat(converter.canWrite(Product.class, null)).isTrue();
    }

    @Test
    void shouldSupportWriteForAnyClass_WhenMediaTypeIsApplicationXml() {
        assertThat(converter.canWrite(Product.class, MediaType.APPLICATION_XML)).isTrue();
    }

    @Test
    void shouldNotSupportWrite_WhenMediaTypeIsJson() {
        assertThat(converter.canWrite(Product.class, MediaType.APPLICATION_JSON)).isFalse();
    }

    @Test
    void shouldSerializePojoToXml_WhenWriting() throws Exception {
        Product product = new Product("Widget", 9.99);

        MockHttpServletResponse response = new MockHttpServletResponse();
        converter.write(product, MediaType.APPLICATION_XML, response);

        String xml = response.getContentAsString();
        assertThat(xml).contains("<Product");
        assertThat(xml).contains("<name>Widget</name>");
        assertThat(xml).contains("<price>9.99</price>");
        assertThat(response.getContentType()).startsWith("application/xml");
    }

    @Test
    void shouldSetTextXmlContentType_WhenMediaTypeIsTextXml() throws Exception {
        Product product = new Product("Gadget", 19.99);

        MockHttpServletResponse response = new MockHttpServletResponse();
        converter.write(product, MediaType.TEXT_XML, response);

        assertThat(response.getContentType()).startsWith("text/xml");
    }

    @Test
    void shouldDeserializeXmlToPojo_WhenReading() throws Exception {
        String xml = "<Product><name>Widget</name><price>9.99</price></Product>";

        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setContent(xml.getBytes(StandardCharsets.UTF_8));
        request.setContentType("application/xml");

        Object result = converter.read(Product.class, request);

        assertThat(result).isInstanceOf(Product.class);
        Product product = (Product) result;
        assertThat(product.getName()).isEqualTo("Widget");
        assertThat(product.getPrice()).isEqualTo(9.99);
    }

    @Test
    void shouldReturnApplicationXmlAndTextXml_WhenGettingSupportedMediaTypes() {
        assertThat(converter.getSupportedMediaTypes())
                .containsExactly(MediaType.APPLICATION_XML, MediaType.TEXT_XML);
    }

    @Test
    void shouldRoundTrip_WhenSerializingAndDeserializing() throws Exception {
        Product original = new Product("Roundtrip Item", 42.0);

        MockHttpServletResponse response = new MockHttpServletResponse();
        converter.write(original, MediaType.APPLICATION_XML, response);
        String xml = response.getContentAsString();

        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setContent(xml.getBytes(StandardCharsets.UTF_8));
        request.setContentType("application/xml");

        Product deserialized = (Product) converter.read(Product.class, request);
        assertThat(deserialized.getName()).isEqualTo("Roundtrip Item");
        assertThat(deserialized.getPrice()).isEqualTo(42.0);
    }

    public static class Product {
        private String name;
        private double price;

        public Product() {}

        public Product(String name, double price) {
            this.name = name;
            this.price = price;
        }

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public double getPrice() { return price; }
        public void setPrice(double price) { this.price = price; }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/XmlMessageConversionIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.adapter.ResponseBodyReturnValueHandler;
import com.simplespringmvc.adapter.SimpleHandlerAdapter;
import com.simplespringmvc.converter.*;
import com.simplespringmvc.http.ContentNegotiationManager;
import com.simplespringmvc.http.MediaType;
import jakarta.xml.bind.annotation.XmlRootElement;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;

import java.nio.charset.StandardCharsets;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class XmlMessageConversionIntegrationTest {

    private List<HttpMessageConverter> converters;
    private ContentNegotiationManager contentNegotiationManager;

    @BeforeEach
    void setUp() {
        converters = List.of(
                new JacksonMessageConverter(),
                new PlainTextMessageConverter(),
                new Jaxb2XmlMessageConverter(),
                new JacksonXmlMessageConverter()
        );
        contentNegotiationManager = new ContentNegotiationManager();
    }

    @Test
    void shouldSelectJsonConverter_WhenAcceptHeaderIsApplicationJson() {
        HttpMessageConverter selected = findWriteConverter(Product.class, MediaType.APPLICATION_JSON);
        assertThat(selected).isInstanceOf(JacksonMessageConverter.class);
    }

    @Test
    void shouldSelectXmlConverter_WhenAcceptHeaderIsApplicationXml() {
        HttpMessageConverter selected = findWriteConverter(Product.class, MediaType.APPLICATION_XML);
        assertThat(selected).isInstanceOf(JacksonXmlMessageConverter.class);
    }

    @Test
    void shouldSelectJaxbConverter_WhenAcceptIsXmlAndClassHasXmlRootElement() {
        HttpMessageConverter selected = findWriteConverter(JaxbItem.class, MediaType.APPLICATION_XML);
        assertThat(selected).isInstanceOf(Jaxb2XmlMessageConverter.class);
    }

    @Test
    void shouldSelectJsonConverter_WhenAcceptIsWildcard() {
        HttpMessageConverter selected = findWriteConverter(Product.class, MediaType.ALL);
        assertThat(selected).isInstanceOf(JacksonMessageConverter.class);
    }

    @Test
    void shouldDeserializeXmlRequestBody_WhenContentTypeIsApplicationXml() throws Exception {
        String xml = "<Product><name>Widget</name><price>9.99</price></Product>";

        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setContent(xml.getBytes(StandardCharsets.UTF_8));
        request.setContentType("application/xml");

        MediaType contentType = MediaType.parseMediaType(request.getContentType());
        HttpMessageConverter readConverter = findReadConverter(Product.class, contentType);

        assertThat(readConverter).isNotNull();
        Product product = (Product) readConverter.read(Product.class, request);
        assertThat(product.getName()).isEqualTo("Widget");
        assertThat(product.getPrice()).isEqualTo(9.99);
    }

    @Test
    void shouldDeserializeJaxbXmlRequestBody_WhenContentTypeIsApplicationXml() throws Exception {
        String xml = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>"
                + "<jaxbItem><id>42</id><description>Test Item</description></jaxbItem>";

        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setContent(xml.getBytes(StandardCharsets.UTF_8));
        request.setContentType("application/xml");

        MediaType contentType = MediaType.parseMediaType(request.getContentType());
        HttpMessageConverter readConverter = findReadConverter(JaxbItem.class, contentType);

        assertThat(readConverter).isInstanceOf(Jaxb2XmlMessageConverter.class);
        JaxbItem item = (JaxbItem) readConverter.read(JaxbItem.class, request);
        assertThat(item.getId()).isEqualTo(42);
        assertThat(item.getDescription()).isEqualTo("Test Item");
    }

    @Test
    void shouldSerializeAsXml_WhenNegotiatedMediaTypeIsApplicationXml() throws Exception {
        Product product = new Product("Gadget", 29.99);
        MockHttpServletResponse response = new MockHttpServletResponse();

        HttpMessageConverter writeConverter = findWriteConverter(Product.class, MediaType.APPLICATION_XML);
        writeConverter.write(product, MediaType.APPLICATION_XML, response);

        assertThat(response.getContentType()).startsWith("application/xml");
        String body = response.getContentAsString();
        assertThat(body).contains("<name>Gadget</name>");
        assertThat(body).contains("<price>29.99</price>");
    }

    @Test
    void shouldSerializeAsJson_WhenNegotiatedMediaTypeIsApplicationJson() throws Exception {
        Product product = new Product("Gadget", 29.99);
        MockHttpServletResponse response = new MockHttpServletResponse();

        HttpMessageConverter writeConverter = findWriteConverter(Product.class, MediaType.APPLICATION_JSON);
        writeConverter.write(product, MediaType.APPLICATION_JSON, response);

        assertThat(response.getContentType()).startsWith("application/json");
        String body = response.getContentAsString();
        assertThat(body).contains("\"name\":\"Gadget\"");
        assertThat(body).contains("\"price\":29.99");
    }

    @Test
    void shouldRegisterXmlConverters_WhenAdapterIsCreated() {
        SimpleHandlerAdapter adapter = new SimpleHandlerAdapter();
        List<HttpMessageConverter> registered = adapter.getMessageConverters();

        assertThat(registered).hasSize(4);
        assertThat(registered.get(0)).isInstanceOf(JacksonMessageConverter.class);
        assertThat(registered.get(1)).isInstanceOf(PlainTextMessageConverter.class);
        assertThat(registered.get(2)).isInstanceOf(Jaxb2XmlMessageConverter.class);
        assertThat(registered.get(3)).isInstanceOf(JacksonXmlMessageConverter.class);
    }

    @Test
    void shouldSerializeAsTextXml_WhenNegotiatedMediaTypeIsTextXml() throws Exception {
        Product product = new Product("Item", 5.0);
        MockHttpServletResponse response = new MockHttpServletResponse();

        HttpMessageConverter writeConverter = findWriteConverter(Product.class, MediaType.TEXT_XML);
        writeConverter.write(product, MediaType.TEXT_XML, response);

        assertThat(response.getContentType()).startsWith("text/xml");
        assertThat(response.getContentAsString()).contains("<name>Item</name>");
    }

    private HttpMessageConverter findWriteConverter(Class<?> clazz, MediaType mediaType) {
        for (HttpMessageConverter converter : converters) {
            if (converter.canWrite(clazz, mediaType)) {
                return converter;
            }
        }
        return null;
    }

    private HttpMessageConverter findReadConverter(Class<?> clazz, MediaType mediaType) {
        for (HttpMessageConverter converter : converters) {
            if (converter.canRead(clazz, mediaType)) {
                return converter;
            }
        }
        return null;
    }

    public static class Product {
        private String name;
        private double price;

        public Product() {}

        public Product(String name, double price) {
            this.name = name;
            this.price = price;
        }

        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public double getPrice() { return price; }
        public void setPrice(double price) { this.price = price; }
    }

    @XmlRootElement
    public static class JaxbItem {
        private int id;
        private String description;

        public JaxbItem() {}

        public int getId() { return id; }
        public void setId(int id) { this.id = id; }
        public String getDescription() { return description; }
        public void setDescription(String description) { this.description = description; }
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **JAXB (Jakarta XML Binding)** | Standards-based XML serialization requiring `@XmlRootElement` annotation — schema-first, namespace-aware |
| **Jackson XML** | Annotation-free XML serialization using `XmlMapper` — "JSON but in XML" for REST APIs |
| **JAXBContext caching** | Expensive-to-create but thread-safe contexts are cached per class; cheap Marshallers are created fresh per request |
| **Converter ordering** | More specific converters (JAXB) come before generic ones (Jackson XML) in the chain; first match wins |
| **Open/Closed Principle** | Adding a new wire format (XML, YAML, Protobuf) requires only new converter implementations — zero changes to the dispatch pipeline |

**Next: Chapter 27 — Async Request Handling (DeferredResult, Callable)** — Allow controller methods to return `DeferredResult<T>` or `Callable<T>` to process requests asynchronously, freeing the Servlet thread while background work completes.
