# Chapter 25: Multipart File Upload

> Line references based on commit `11ab0b4351` of the Spring Framework repository.

## Build Challenge

| Current State | Limitation | Objective |
|---------------|------------|-----------|
| Controllers can receive query params, path variables, JSON bodies, and form data — but the framework has no way to handle `multipart/form-data` requests for file uploads | A POST with an attached file hits the controller, but there's no `MultipartFile` abstraction or Servlet multipart configuration — the uploaded bytes are inaccessible | Build a `MultipartResolver` strategy that detects and wraps multipart requests, a `MultipartFile` interface that abstracts uploaded files, and a `MultipartFileArgumentResolver` that injects files into controller method parameters |

---

## 25.1 The Integration Point

The multipart feature plugs into the system at **three seams** — the DispatcherServlet dispatch pipeline, the handler adapter's resolver chain, and the embedded Tomcat's servlet configuration.

**Modifying:** `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java`
**Change:** Add multipart request detection and wrapping at the top of `doDispatch()`, before handler lookup

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
    HandlerExecutionChain mappedHandler = null;
    ModelAndView mv = null;
    Exception dispatchException = null;
    // ch25: Track whether we wrapped the request for multipart cleanup
    boolean multipartRequestParsed = false;
    HttpServletRequest processedRequest = request;

    // ch25: Step 0a — Check for multipart request and wrap if needed.
    // This must happen BEFORE handler lookup so argument resolvers
    // see the MultipartHttpServletRequest wrapper.
    //
    // Maps to: DispatcherServlet.checkMultipart() (line 1105)
    if (multipartResolver != null && multipartResolver.isMultipart(request)) {
        processedRequest = multipartResolver.resolveMultipart(request);
        multipartRequestParsed = (processedRequest != request);
    }

    // ... rest of dispatch uses processedRequest instead of request
```

**Modifying:** `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java`
**Change:** Register `MultipartFileArgumentResolver` before `RequestParamArgumentResolver`

```java
argumentResolvers.addResolver(new PathVariableArgumentResolver(conversionService));
// ch25: MultipartFileArgumentResolver BEFORE RequestParamArgumentResolver.
// Both match @RequestParam, but MultipartFileArgumentResolver only matches
// MultipartFile type. It must come first so that @RequestParam MultipartFile
// parameters are resolved by the multipart resolver, not the query param resolver.
argumentResolvers.addResolver(new MultipartFileArgumentResolver());
argumentResolvers.addResolver(new RequestParamArgumentResolver(conversionService));
```

**Modifying:** `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java`
**Change:** Add `enableMultipart()` method that sets `MultipartConfigElement` on the servlet registration

```java
public void enableMultipart(String location, long maxFileSize,
                             long maxRequestSize, int fileSizeThreshold) {
    servletWrapper.setMultipartConfigElement(
            new MultipartConfigElement(location, maxFileSize, maxRequestSize, fileSizeThreshold));
}
```

Two key decisions here:

1. **Multipart wrapping happens before handler lookup** — the `processedRequest` replaces the original request for the entire dispatch pipeline. This ensures every downstream component (interceptors, argument resolvers, return value handlers) sees the `MultipartHttpServletRequest` wrapper.

2. **A dedicated resolver rather than extending `RequestParamArgumentResolver`** — in the real framework, `RequestParamMethodArgumentResolver` handles both query params and multipart files via delegation to `MultipartResolutionDelegate`. We keep them separate for clarity, but register the multipart resolver first so it takes priority for `MultipartFile` parameters.

This connects the **Servlet multipart infrastructure** to the **argument resolution pipeline**. To make it work, we need to build:
- `MultipartFile` — the abstraction controllers use to access uploaded files
- `MultipartResolver` / `StandardServletMultipartResolver` — the strategy that detects and wraps multipart requests
- `MultipartHttpServletRequest` — the wrapper that parses Parts into MultipartFiles
- `StandardMultipartFile` — the adapter from Servlet `Part` to `MultipartFile`
- `MultipartFileArgumentResolver` — the bridge between `@RequestParam MultipartFile` and the wrapper

## 25.2 The MultipartFile Interface

**New file:** `src/main/java/com/simplespringmvc/multipart/MultipartFile.java`

```java
package com.simplespringmvc.multipart;

import java.io.IOException;
import java.io.InputStream;

public interface MultipartFile {

    String getName();

    String getOriginalFilename();

    String getContentType();

    boolean isEmpty();

    long getSize();

    byte[] getBytes() throws IOException;

    InputStream getInputStream() throws IOException;
}
```

## 25.3 The MultipartResolver Strategy

**New file:** `src/main/java/com/simplespringmvc/multipart/MultipartResolver.java`

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.HttpServletRequest;

public interface MultipartResolver {

    boolean isMultipart(HttpServletRequest request);

    MultipartHttpServletRequest resolveMultipart(HttpServletRequest request);
}
```

**New file:** `src/main/java/com/simplespringmvc/multipart/StandardServletMultipartResolver.java`

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.HttpServletRequest;
import java.util.Locale;

public class StandardServletMultipartResolver implements MultipartResolver {

    @Override
    public boolean isMultipart(HttpServletRequest request) {
        String contentType = request.getContentType();
        return contentType != null
                && contentType.toLowerCase(Locale.ROOT).startsWith("multipart/");
    }

    @Override
    public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) {
        return new MultipartHttpServletRequest(request);
    }
}
```

## 25.4 The Request Wrapper and File Adapter

**New file:** `src/main/java/com/simplespringmvc/multipart/StandardMultipartFile.java`

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.Part;
import java.io.IOException;
import java.io.InputStream;

public class StandardMultipartFile implements MultipartFile {

    private final Part part;

    public StandardMultipartFile(Part part) {
        this.part = part;
    }

    @Override
    public String getName() { return part.getName(); }

    @Override
    public String getOriginalFilename() { return part.getSubmittedFileName(); }

    @Override
    public String getContentType() { return part.getContentType(); }

    @Override
    public boolean isEmpty() { return part.getSize() == 0; }

    @Override
    public long getSize() { return part.getSize(); }

    @Override
    public byte[] getBytes() throws IOException {
        return part.getInputStream().readAllBytes();
    }

    @Override
    public InputStream getInputStream() throws IOException {
        return part.getInputStream();
    }
}
```

**New file:** `src/main/java/com/simplespringmvc/multipart/MultipartHttpServletRequest.java`

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletRequestWrapper;
import jakarta.servlet.http.Part;
import java.util.*;

public class MultipartHttpServletRequest extends HttpServletRequestWrapper {

    private final Map<String, List<MultipartFile>> multipartFiles = new LinkedHashMap<>();

    public MultipartHttpServletRequest(HttpServletRequest request) {
        super(request);
        try {
            for (Part part : request.getParts()) {
                if (part.getSubmittedFileName() != null) {
                    String name = part.getName();
                    multipartFiles.computeIfAbsent(name, k -> new ArrayList<>())
                            .add(new StandardMultipartFile(part));
                }
            }
        } catch (Exception ex) {
            throw new MultipartException("Failed to parse multipart request", ex);
        }
    }

    public MultipartFile getFile(String name) {
        List<MultipartFile> files = multipartFiles.get(name);
        return (files != null && !files.isEmpty()) ? files.get(0) : null;
    }

    public List<MultipartFile> getFiles(String name) {
        List<MultipartFile> files = multipartFiles.get(name);
        return files != null ? Collections.unmodifiableList(files) : Collections.emptyList();
    }

    public Map<String, List<MultipartFile>> getMultipartFiles() {
        return Collections.unmodifiableMap(multipartFiles);
    }

    public boolean hasFiles() { return !multipartFiles.isEmpty(); }
}
```

## 25.5 The MultipartFileArgumentResolver

**New file:** `src/main/java/com/simplespringmvc/adapter/MultipartFileArgumentResolver.java`

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.multipart.MultipartFile;
import com.simplespringmvc.multipart.MultipartHttpServletRequest;
import jakarta.servlet.http.HttpServletRequest;

public class MultipartFileArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return MultipartFile.class.isAssignableFrom(parameter.getParameterType())
                && parameter.hasParameterAnnotation(RequestParam.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request)
            throws Exception {
        String parameterName = getParameterName(parameter);
        MultipartHttpServletRequest multipartRequest = findMultipartRequest(request);

        if (multipartRequest == null) {
            throw new IllegalStateException(
                    "Expected a multipart request for parameter '" + parameterName
                            + "' but the current request is not a multipart request.");
        }

        MultipartFile file = multipartRequest.getFile(parameterName);
        if (file == null) {
            throw new IllegalStateException(
                    "Required multipart file '" + parameterName + "' is not present");
        }
        return file;
    }

    private MultipartHttpServletRequest findMultipartRequest(HttpServletRequest request) {
        if (request instanceof MultipartHttpServletRequest mpr) return mpr;
        while (request instanceof jakarta.servlet.http.HttpServletRequestWrapper wrapper) {
            request = (HttpServletRequest) wrapper.getRequest();
            if (request instanceof MultipartHttpServletRequest mpr) return mpr;
        }
        return null;
    }

    private String getParameterName(MethodParameter parameter) {
        RequestParam annotation = parameter.getParameterAnnotation(RequestParam.class);
        return (annotation != null && !annotation.value().isEmpty())
                ? annotation.value() : parameter.getParameterName();
    }
}
```

## 25.6 Try It Yourself

<details>
<summary>Challenge: Implement the file-vs-form-field distinction in MultipartHttpServletRequest</summary>

A multipart request can contain both files and regular form fields. Both arrive as `Part` objects from the Servlet API. How do you distinguish them?

**Hint:** Check what `Part.getSubmittedFileName()` returns for a regular form field.

```java
// In the constructor:
for (Part part : request.getParts()) {
    // A Part is a file upload if it has a submitted filename.
    // Form fields (text inputs) have null submitted filename.
    if (part.getSubmittedFileName() != null) {
        String name = part.getName();
        multipartFiles.computeIfAbsent(name, k -> new ArrayList<>())
                .add(new StandardMultipartFile(part));
    }
}
```

The key insight: `getSubmittedFileName()` returns `null` for form fields because only `<input type="file">` elements include a filename in the `Content-Disposition` header.

</details>

<details>
<summary>Challenge: Wire multipart into an EmbeddedTomcat-based application</summary>

You have a controller:
```java
@PostMapping("/upload")
@ResponseBody
public String upload(@RequestParam("file") MultipartFile file) {
    return "Received: " + file.getOriginalFilename();
}
```

What three setup steps are needed before this works?

```java
// 1. Configure the DispatcherServlet with a MultipartResolver
servlet.setMultipartResolver(new StandardServletMultipartResolver());

// 2. Enable multipart support in Tomcat (sets MultipartConfigElement)
tomcat.enableMultipart();

// 3. Start the server
tomcat.start();
```

Without step 1, the request won't be wrapped. Without step 2, `request.getParts()` throws `IllegalStateException`.

</details>

## 25.7 Tests

### Unit Tests

**New file:** `src/test/java/com/simplespringmvc/multipart/StandardMultipartFileTest.java`

```java
@Test
@DisplayName("should return parameter name from Part")
void shouldReturnName() {
    when(mockPart.getName()).thenReturn("file");
    assertThat(multipartFile.getName()).isEqualTo("file");
}

@Test
@DisplayName("should return original filename from Part's submitted filename")
void shouldReturnOriginalFilename() {
    when(mockPart.getSubmittedFileName()).thenReturn("photo.jpg");
    assertThat(multipartFile.getOriginalFilename()).isEqualTo("photo.jpg");
}

@Test
@DisplayName("should detect empty file when size is 0")
void shouldDetectEmpty_WhenSizeIsZero() {
    when(mockPart.getSize()).thenReturn(0L);
    assertThat(multipartFile.isEmpty()).isTrue();
}
```

**New file:** `src/test/java/com/simplespringmvc/multipart/MultipartHttpServletRequestTest.java`

```java
@Test
@DisplayName("should parse single file from multipart request")
void shouldParseSingleFile() throws Exception {
    Part filePart = createFilePart("document", "report.pdf", "application/pdf", "PDF content".getBytes());
    HttpServletRequest mockRequest = createRequestWithParts(filePart);

    MultipartHttpServletRequest request = new MultipartHttpServletRequest(mockRequest);

    MultipartFile file = request.getFile("document");
    assertThat(file).isNotNull();
    assertThat(file.getName()).isEqualTo("document");
    assertThat(file.getOriginalFilename()).isEqualTo("report.pdf");
}

@Test
@DisplayName("should skip form field parts (no submitted filename)")
void shouldSkipFormFields() throws Exception {
    Part filePart = createFilePart("file", "test.txt", "text/plain", "content".getBytes());
    Part formField = createFormFieldPart("name", "John");
    HttpServletRequest mockRequest = createRequestWithParts(filePart, formField);

    MultipartHttpServletRequest request = new MultipartHttpServletRequest(mockRequest);

    assertThat(request.getMultipartFiles()).hasSize(1);
    assertThat(request.getFile("name")).isNull();
}
```

### Integration Tests

**New file:** `src/test/java/com/simplespringmvc/integration/MultipartFileUploadIntegrationTest.java`

```java
@Test
@DisplayName("should upload file and receive metadata in response")
void shouldUploadFile_WhenSingleFilePosted() throws Exception {
    String boundary = UUID.randomUUID().toString();
    String body = buildMultipartBody(boundary, "file", "hello.txt", "text/plain", "Hello, World!");

    HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + "/upload"))
            .header("Content-Type", "multipart/form-data; boundary=" + boundary)
            .POST(HttpRequest.BodyPublishers.ofString(body))
            .build();

    HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());

    assertThat(response.statusCode()).isEqualTo(200);
    assertThat(response.body()).contains("name=hello.txt");
    assertThat(response.body()).contains("content=Hello, World!");
}

@Test
@DisplayName("should upload file with additional form parameter")
void shouldUploadFile_WhenFileAndParamPosted() throws Exception {
    // ... multipart body with both file and text field parts
    assertThat(response.body()).contains("file=doc.pdf");
    assertThat(response.body()).contains("desc=My important document");
}
```

**Run:** `./gradlew test` — expected: all tests pass (including prior features' tests)

---

## 25.8 Why This Works

> ★ **Insight** -------------------------------------------
> - **Why a strategy interface for multipart?** The `MultipartResolver` strategy separates multipart *detection* from *parsing*. Historically, Spring had two implementations — `CommonsMultipartResolver` (Apache Commons FileUpload) and `StandardServletMultipartResolver` (Servlet 3.0 Part API). Spring 6 dropped Commons FileUpload, but the strategy still matters: it lets the DispatcherServlet remain agnostic about multipart implementation, and allows disabling multipart support entirely by not setting a resolver.
> - **When you might skip this:** If your API only accepts JSON bodies (`@RequestBody`), you don't need multipart at all. Even when you do, you could bypass `MultipartFile` and use `jakarta.servlet.http.Part` directly — but then your code depends on the Servlet API, which breaks portability to reactive stacks.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **Why wrapping the request instead of extracting files separately?** The `HttpServletRequestWrapper` pattern lets the multipart request flow through the entire dispatch pipeline transparently. Handler mappings, interceptors, and argument resolvers all receive the same `HttpServletRequest` interface — they don't need to know it's been wrapped. This is the Decorator pattern in action.
> - **The real framework takes this further:** `StandardMultipartHttpServletRequest` also overrides `getParameter()` and `getParameterMap()` to include multipart form field values, making them accessible via the standard Servlet API. Our simplified version keeps files and form fields separate.
> -----------------------------------------------------------

> ★ **Insight** -------------------------------------------
> - **The two-layer configuration:** Multipart requires configuration at *both* layers: (1) the Servlet container must know to parse multipart bodies (`MultipartConfigElement` on the servlet registration), and (2) the framework must know to wrap the request (`MultipartResolver` on the DispatcherServlet). Missing either one causes a different failure — no `MultipartConfigElement` means `getParts()` throws; no `MultipartResolver` means the request isn't wrapped and argument resolution fails.
> -----------------------------------------------------------

## 25.9 What We Enhanced

| Aspect | Before (ch05) | Current (ch25) | Real Framework |
|--------|---------------|----------------|----------------|
| `@RequestParam` handling | Only query string and form data (`request.getParameter()`) | Also handles `MultipartFile` parameters via dedicated resolver | `RequestParamMethodArgumentResolver` delegates to `MultipartResolutionDelegate` for file types (`RequestParamMethodArgumentResolver.java:128`) |
| Request wrapping | Request flows through dispatch pipeline unmodified | Multipart requests are wrapped in `MultipartHttpServletRequest` before handler lookup | `DispatcherServlet.checkMultipart()` wraps via `MultipartResolver` (`DispatcherServlet.java:1105`) |
| Servlet configuration | Async support only | Async + multipart config via `MultipartConfigElement` | Spring Boot's `MultipartAutoConfiguration` creates `MultipartConfigElement` with configurable properties (`MultipartAutoConfiguration.java`) |
| Simplification removed | "No multipart handling" listed in DispatcherServlet Javadoc | Multipart now supported | Full multipart with lazy parsing, cleanup, Part collections, arrays |

## 25.10 Connection to Real Framework

| Simplified Code | Real Framework Code | File:Line | Key Difference |
|----------------|--------------------|-----------| ---------------|
| `MultipartFile` | `MultipartFile` | `MultipartFile.java:42` | Real adds `transferTo(File)`, `transferTo(Path)`, `getResource()` |
| `MultipartResolver` | `MultipartResolver` | `MultipartResolver.java:43` | Real adds `cleanupMultipart()` for post-request cleanup |
| `StandardServletMultipartResolver` | `StandardServletMultipartResolver` | `StandardServletMultipartResolver.java:50` | Real supports `resolveLazily` flag and strict compliance mode |
| `MultipartHttpServletRequest` | `StandardMultipartHttpServletRequest` | `StandardMultipartHttpServletRequest.java:56` | Real extends `AbstractMultipartHttpServletRequest`, overrides `getParameter()`, supports lazy Part parsing |
| `StandardMultipartFile` | `StandardMultipartFile` (inner class) | `StandardMultipartHttpServletRequest.java:188` | Real strips path info from filenames |
| `MultipartFileArgumentResolver` | `MultipartResolutionDelegate` | `MultipartResolutionDelegate.java:60` | Real handles `List<MultipartFile>`, `MultipartFile[]`, `Part`, `List<Part>`, `Part[]` |
| `doDispatch()` multipart check | `checkMultipart()` | `DispatcherServlet.java:1105` | Real has `cleanupMultipart()` in finally block |
| `enableMultipart()` | `MultipartAutoConfiguration` | `MultipartAutoConfiguration.java` | Real uses Spring Boot auto-config with `spring.servlet.multipart.*` properties |

## 25.11 Complete Code

### Production Code

#### File: `src/main/java/com/simplespringmvc/multipart/MultipartFile.java` [NEW]

```java
package com.simplespringmvc.multipart;

import java.io.IOException;
import java.io.InputStream;

/**
 * A representation of an uploaded file received in a multipart request.
 *
 * Maps to: {@code org.springframework.web.multipart.MultipartFile}
 */
public interface MultipartFile {

    String getName();

    String getOriginalFilename();

    String getContentType();

    boolean isEmpty();

    long getSize();

    byte[] getBytes() throws IOException;

    InputStream getInputStream() throws IOException;
}
```

#### File: `src/main/java/com/simplespringmvc/multipart/MultipartResolver.java` [NEW]

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.HttpServletRequest;

/**
 * Strategy interface for multipart file upload resolution.
 *
 * Maps to: {@code org.springframework.web.multipart.MultipartResolver}
 */
public interface MultipartResolver {

    boolean isMultipart(HttpServletRequest request);

    MultipartHttpServletRequest resolveMultipart(HttpServletRequest request);
}
```

#### File: `src/main/java/com/simplespringmvc/multipart/StandardServletMultipartResolver.java` [NEW]

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.HttpServletRequest;

import java.util.Locale;

/**
 * Standard implementation of {@link MultipartResolver} that uses the
 * Servlet 3.0 {@code Part} API for multipart parsing.
 *
 * Maps to: {@code org.springframework.web.multipart.support.StandardServletMultipartResolver}
 */
public class StandardServletMultipartResolver implements MultipartResolver {

    @Override
    public boolean isMultipart(HttpServletRequest request) {
        String contentType = request.getContentType();
        return contentType != null
                && contentType.toLowerCase(Locale.ROOT).startsWith("multipart/");
    }

    @Override
    public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) {
        return new MultipartHttpServletRequest(request);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/multipart/StandardMultipartFile.java` [NEW]

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.Part;

import java.io.IOException;
import java.io.InputStream;

/**
 * {@link MultipartFile} implementation that wraps a Servlet 3.0 {@code Part}.
 *
 * Maps to: {@code StandardMultipartHttpServletRequest.StandardMultipartFile} (inner class)
 */
public class StandardMultipartFile implements MultipartFile {

    private final Part part;

    public StandardMultipartFile(Part part) {
        this.part = part;
    }

    @Override
    public String getName() {
        return part.getName();
    }

    @Override
    public String getOriginalFilename() {
        return part.getSubmittedFileName();
    }

    @Override
    public String getContentType() {
        return part.getContentType();
    }

    @Override
    public boolean isEmpty() {
        return part.getSize() == 0;
    }

    @Override
    public long getSize() {
        return part.getSize();
    }

    @Override
    public byte[] getBytes() throws IOException {
        return part.getInputStream().readAllBytes();
    }

    @Override
    public InputStream getInputStream() throws IOException {
        return part.getInputStream();
    }

    public Part getPart() {
        return part;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/multipart/MultipartHttpServletRequest.java` [NEW]

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletRequestWrapper;
import jakarta.servlet.http.Part;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * {@link HttpServletRequestWrapper} that wraps a multipart request and provides
 * access to uploaded files as {@link MultipartFile} objects.
 *
 * Maps to: {@code org.springframework.web.multipart.support.StandardMultipartHttpServletRequest}
 */
public class MultipartHttpServletRequest extends HttpServletRequestWrapper {

    private final Map<String, List<MultipartFile>> multipartFiles = new LinkedHashMap<>();

    public MultipartHttpServletRequest(HttpServletRequest request) {
        super(request);
        try {
            for (Part part : request.getParts()) {
                if (part.getSubmittedFileName() != null) {
                    String name = part.getName();
                    multipartFiles.computeIfAbsent(name, k -> new ArrayList<>())
                            .add(new StandardMultipartFile(part));
                }
            }
        } catch (Exception ex) {
            throw new MultipartException("Failed to parse multipart request", ex);
        }
    }

    public MultipartFile getFile(String name) {
        List<MultipartFile> files = multipartFiles.get(name);
        return (files != null && !files.isEmpty()) ? files.get(0) : null;
    }

    public List<MultipartFile> getFiles(String name) {
        List<MultipartFile> files = multipartFiles.get(name);
        return files != null ? Collections.unmodifiableList(files) : Collections.emptyList();
    }

    public Map<String, List<MultipartFile>> getMultipartFiles() {
        return Collections.unmodifiableMap(multipartFiles);
    }

    public boolean hasFiles() {
        return !multipartFiles.isEmpty();
    }
}
```

#### File: `src/main/java/com/simplespringmvc/multipart/MultipartException.java` [NEW]

```java
package com.simplespringmvc.multipart;

/**
 * Exception thrown when multipart resolution fails.
 *
 * Maps to: {@code org.springframework.web.multipart.MultipartException}
 */
public class MultipartException extends RuntimeException {

    public MultipartException(String message, Throwable cause) {
        super(message, cause);
    }

    public MultipartException(String message) {
        super(message);
    }
}
```

#### File: `src/main/java/com/simplespringmvc/adapter/MultipartFileArgumentResolver.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.multipart.MultipartFile;
import com.simplespringmvc.multipart.MultipartHttpServletRequest;
import jakarta.servlet.http.HttpServletRequest;

/**
 * Resolves method arguments of type {@link MultipartFile} annotated with
 * {@link RequestParam} by extracting uploaded files from the multipart request.
 *
 * Maps to: {@code MultipartResolutionDelegate.resolveMultipartArgument()}
 */
public class MultipartFileArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return MultipartFile.class.isAssignableFrom(parameter.getParameterType())
                && parameter.hasParameterAnnotation(RequestParam.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, HttpServletRequest request)
            throws Exception {
        String parameterName = getParameterName(parameter);
        MultipartHttpServletRequest multipartRequest = findMultipartRequest(request);

        if (multipartRequest == null) {
            throw new IllegalStateException(
                    "Expected a multipart request for parameter '" + parameterName
                            + "' but the current request is not a multipart request. "
                            + "Ensure a MultipartResolver is configured.");
        }

        MultipartFile file = multipartRequest.getFile(parameterName);
        if (file == null) {
            throw new IllegalStateException(
                    "Required multipart file '" + parameterName + "' is not present");
        }

        return file;
    }

    private MultipartHttpServletRequest findMultipartRequest(HttpServletRequest request) {
        if (request instanceof MultipartHttpServletRequest mpr) {
            return mpr;
        }
        while (request instanceof jakarta.servlet.http.HttpServletRequestWrapper wrapper) {
            request = (HttpServletRequest) wrapper.getRequest();
            if (request instanceof MultipartHttpServletRequest mpr) {
                return mpr;
            }
        }
        return null;
    }

    private String getParameterName(MethodParameter parameter) {
        RequestParam annotation = parameter.getParameterAnnotation(RequestParam.class);
        String name = (annotation != null && !annotation.value().isEmpty())
                ? annotation.value()
                : parameter.getParameterName();
        return name;
    }
}
```

#### File: `src/main/java/com/simplespringmvc/servlet/SimpleDispatcherServlet.java` [MODIFIED]

Changes in ch25:
- Added `multipartResolver` field
- Added multipart check at top of `doDispatch()` — wraps request before handler lookup
- All dispatch pipeline references changed from `request` to `processedRequest`
- Added `setMultipartResolver()` / `getMultipartResolver()` accessors
- Removed "No multipart handling" from simplification notes

#### File: `src/main/java/com/simplespringmvc/adapter/SimpleHandlerAdapter.java` [MODIFIED]

Changes in ch25:
- Added `MultipartFileArgumentResolver` registration before `RequestParamArgumentResolver` in both constructors

#### File: `src/main/java/com/simplespringmvc/server/EmbeddedTomcat.java` [MODIFIED]

Changes in ch25:
- Added `servletWrapper` field to store the Tomcat Wrapper reference
- Added `enableMultipart(location, maxFileSize, maxRequestSize, fileSizeThreshold)` method
- Added `enableMultipart()` convenience method with sensible defaults

### Test Code

#### File: `src/test/java/com/simplespringmvc/multipart/StandardMultipartFileTest.java` [NEW]

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.Part;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class StandardMultipartFileTest {

    private Part mockPart;
    private StandardMultipartFile multipartFile;

    @BeforeEach
    void setUp() {
        mockPart = mock(Part.class);
        multipartFile = new StandardMultipartFile(mockPart);
    }

    @Nested
    @DisplayName("Metadata delegation")
    class MetadataDelegation {

        @Test
        @DisplayName("should return parameter name from Part")
        void shouldReturnName() {
            when(mockPart.getName()).thenReturn("file");
            assertThat(multipartFile.getName()).isEqualTo("file");
        }

        @Test
        @DisplayName("should return original filename from Part's submitted filename")
        void shouldReturnOriginalFilename() {
            when(mockPart.getSubmittedFileName()).thenReturn("photo.jpg");
            assertThat(multipartFile.getOriginalFilename()).isEqualTo("photo.jpg");
        }

        @Test
        @DisplayName("should return content type from Part")
        void shouldReturnContentType() {
            when(mockPart.getContentType()).thenReturn("image/jpeg");
            assertThat(multipartFile.getContentType()).isEqualTo("image/jpeg");
        }

        @Test
        @DisplayName("should return size from Part")
        void shouldReturnSize() {
            when(mockPart.getSize()).thenReturn(12345L);
            assertThat(multipartFile.getSize()).isEqualTo(12345L);
        }
    }

    @Nested
    @DisplayName("Empty detection")
    class EmptyDetection {

        @Test
        @DisplayName("should detect empty file when size is 0")
        void shouldDetectEmpty_WhenSizeIsZero() {
            when(mockPart.getSize()).thenReturn(0L);
            assertThat(multipartFile.isEmpty()).isTrue();
        }

        @Test
        @DisplayName("should detect non-empty file when size is positive")
        void shouldDetectNonEmpty_WhenSizeIsPositive() {
            when(mockPart.getSize()).thenReturn(100L);
            assertThat(multipartFile.isEmpty()).isFalse();
        }
    }

    @Nested
    @DisplayName("Content access")
    class ContentAccess {

        @Test
        @DisplayName("should return bytes from Part's input stream")
        void shouldReturnBytes() throws IOException {
            byte[] content = "Hello, World!".getBytes(StandardCharsets.UTF_8);
            when(mockPart.getInputStream()).thenReturn(new ByteArrayInputStream(content));
            assertThat(multipartFile.getBytes()).isEqualTo(content);
        }

        @Test
        @DisplayName("should return input stream from Part")
        void shouldReturnInputStream() throws IOException {
            byte[] content = "test content".getBytes(StandardCharsets.UTF_8);
            ByteArrayInputStream stream = new ByteArrayInputStream(content);
            when(mockPart.getInputStream()).thenReturn(stream);
            assertThat(multipartFile.getInputStream()).isSameAs(stream);
        }
    }

    @Test
    @DisplayName("should expose underlying Part")
    void shouldExposePart() {
        assertThat(multipartFile.getPart()).isSameAs(mockPart);
    }
}
```

#### File: `src/test/java/com/simplespringmvc/multipart/StandardServletMultipartResolverTest.java` [NEW]

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.HttpServletRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Collections;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class StandardServletMultipartResolverTest {

    private StandardServletMultipartResolver resolver;

    @BeforeEach
    void setUp() {
        resolver = new StandardServletMultipartResolver();
    }

    @Nested
    @DisplayName("isMultipart()")
    class IsMultipart {

        @Test
        @DisplayName("should return true for multipart/form-data content type")
        void shouldReturnTrue_WhenMultipartFormData() {
            HttpServletRequest request = mock(HttpServletRequest.class);
            when(request.getContentType()).thenReturn("multipart/form-data; boundary=----abc123");
            assertThat(resolver.isMultipart(request)).isTrue();
        }

        @Test
        @DisplayName("should return true for multipart/mixed content type")
        void shouldReturnTrue_WhenMultipartMixed() {
            HttpServletRequest request = mock(HttpServletRequest.class);
            when(request.getContentType()).thenReturn("multipart/mixed");
            assertThat(resolver.isMultipart(request)).isTrue();
        }

        @Test
        @DisplayName("should be case insensitive")
        void shouldBeCaseInsensitive() {
            HttpServletRequest request = mock(HttpServletRequest.class);
            when(request.getContentType()).thenReturn("MULTIPART/FORM-DATA");
            assertThat(resolver.isMultipart(request)).isTrue();
        }

        @Test
        @DisplayName("should return false for application/json content type")
        void shouldReturnFalse_WhenNotMultipart() {
            HttpServletRequest request = mock(HttpServletRequest.class);
            when(request.getContentType()).thenReturn("application/json");
            assertThat(resolver.isMultipart(request)).isFalse();
        }

        @Test
        @DisplayName("should return false when content type is null")
        void shouldReturnFalse_WhenContentTypeNull() {
            HttpServletRequest request = mock(HttpServletRequest.class);
            when(request.getContentType()).thenReturn(null);
            assertThat(resolver.isMultipart(request)).isFalse();
        }
    }

    @Nested
    @DisplayName("resolveMultipart()")
    class ResolveMultipart {

        @Test
        @DisplayName("should return MultipartHttpServletRequest wrapper")
        void shouldReturnMultipartWrapper() throws Exception {
            HttpServletRequest mockRequest = mock(HttpServletRequest.class);
            when(mockRequest.getParts()).thenReturn(Collections.emptyList());
            MultipartHttpServletRequest result = resolver.resolveMultipart(mockRequest);
            assertThat(result).isNotNull().isInstanceOf(MultipartHttpServletRequest.class);
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/multipart/MultipartHttpServletRequestTest.java` [NEW]

```java
package com.simplespringmvc.multipart;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.Part;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayInputStream;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class MultipartHttpServletRequestTest {

    private Part createFilePart(String name, String filename, String contentType, byte[] content)
            throws Exception {
        Part part = mock(Part.class);
        when(part.getName()).thenReturn(name);
        when(part.getSubmittedFileName()).thenReturn(filename);
        when(part.getContentType()).thenReturn(contentType);
        when(part.getSize()).thenReturn((long) content.length);
        when(part.getInputStream()).thenReturn(new ByteArrayInputStream(content));
        return part;
    }

    private Part createFormFieldPart(String name, String value) throws Exception {
        Part part = mock(Part.class);
        when(part.getName()).thenReturn(name);
        when(part.getSubmittedFileName()).thenReturn(null);
        when(part.getSize()).thenReturn((long) value.length());
        when(part.getInputStream()).thenReturn(new ByteArrayInputStream(value.getBytes()));
        return part;
    }

    private HttpServletRequest createRequestWithParts(Part... parts) throws Exception {
        HttpServletRequest request = mock(HttpServletRequest.class);
        Collection<Part> partCollection = new ArrayList<>(List.of(parts));
        when(request.getParts()).thenReturn(partCollection);
        when(request.getRequestURI()).thenReturn("/upload");
        when(request.getMethod()).thenReturn("POST");
        return request;
    }

    @Nested
    @DisplayName("File parsing")
    class FileParsing {

        @Test
        @DisplayName("should parse single file from multipart request")
        void shouldParseSingleFile() throws Exception {
            Part filePart = createFilePart("document", "report.pdf",
                    "application/pdf", "PDF content".getBytes());
            HttpServletRequest mockRequest = createRequestWithParts(filePart);
            MultipartHttpServletRequest request = new MultipartHttpServletRequest(mockRequest);

            MultipartFile file = request.getFile("document");
            assertThat(file).isNotNull();
            assertThat(file.getName()).isEqualTo("document");
            assertThat(file.getOriginalFilename()).isEqualTo("report.pdf");
            assertThat(file.getContentType()).isEqualTo("application/pdf");
        }

        @Test
        @DisplayName("should handle multiple files with the same parameter name")
        void shouldHandleMultipleFilesWithSameName() throws Exception {
            Part partA = createFilePart("files", "a.txt", "text/plain", "aaa".getBytes());
            Part partB = createFilePart("files", "b.txt", "text/plain", "bbb".getBytes());
            HttpServletRequest mockRequest = createRequestWithParts(partA, partB);
            MultipartHttpServletRequest request = new MultipartHttpServletRequest(mockRequest);

            assertThat(request.getFile("files").getOriginalFilename()).isEqualTo("a.txt");
            List<MultipartFile> files = request.getFiles("files");
            assertThat(files).hasSize(2);
        }

        @Test
        @DisplayName("should skip form field parts (no submitted filename)")
        void shouldSkipFormFields() throws Exception {
            Part filePart = createFilePart("file", "test.txt", "text/plain", "content".getBytes());
            Part formField = createFormFieldPart("name", "John");
            HttpServletRequest mockRequest = createRequestWithParts(filePart, formField);
            MultipartHttpServletRequest request = new MultipartHttpServletRequest(mockRequest);

            assertThat(request.getMultipartFiles()).hasSize(1);
            assertThat(request.getFile("name")).isNull();
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/adapter/MultipartFileArgumentResolverTest.java` [NEW]

```java
package com.simplespringmvc.adapter;

import com.simplespringmvc.annotation.RequestParam;
import com.simplespringmvc.mapping.MethodParameter;
import com.simplespringmvc.multipart.MultipartFile;
import com.simplespringmvc.multipart.MultipartHttpServletRequest;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.Part;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayInputStream;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class MultipartFileArgumentResolverTest {

    private MultipartFileArgumentResolver resolver;

    @BeforeEach
    void setUp() {
        resolver = new MultipartFileArgumentResolver();
    }

    static void uploadFile(@RequestParam("file") MultipartFile file) {}
    static void regularParam(@RequestParam("name") String name) {}
    static void noAnnotation(MultipartFile file) {}

    @Nested
    @DisplayName("supportsParameter()")
    class SupportsParameter {

        @Test
        @DisplayName("should support MultipartFile with @RequestParam")
        void shouldSupport_WhenMultipartFileWithRequestParam() throws Exception {
            Method method = MultipartFileArgumentResolverTest.class
                    .getDeclaredMethod("uploadFile", MultipartFile.class);
            MethodParameter param = new MethodParameter(method, 0);
            assertThat(resolver.supportsParameter(param)).isTrue();
        }

        @Test
        @DisplayName("should not support String with @RequestParam")
        void shouldNotSupport_WhenStringWithRequestParam() throws Exception {
            Method method = MultipartFileArgumentResolverTest.class
                    .getDeclaredMethod("regularParam", String.class);
            MethodParameter param = new MethodParameter(method, 0);
            assertThat(resolver.supportsParameter(param)).isFalse();
        }

        @Test
        @DisplayName("should not support MultipartFile without @RequestParam")
        void shouldNotSupport_WhenNoAnnotation() throws Exception {
            Method method = MultipartFileArgumentResolverTest.class
                    .getDeclaredMethod("noAnnotation", MultipartFile.class);
            MethodParameter param = new MethodParameter(method, 0);
            assertThat(resolver.supportsParameter(param)).isFalse();
        }
    }

    @Nested
    @DisplayName("resolveArgument()")
    class ResolveArgument {

        @Test
        @DisplayName("should resolve MultipartFile from multipart request")
        void shouldResolveFile_WhenPresent() throws Exception {
            Method method = MultipartFileArgumentResolverTest.class
                    .getDeclaredMethod("uploadFile", MultipartFile.class);
            MethodParameter param = new MethodParameter(method, 0);

            Part filePart = mock(Part.class);
            when(filePart.getName()).thenReturn("file");
            when(filePart.getSubmittedFileName()).thenReturn("test.txt");
            when(filePart.getContentType()).thenReturn("text/plain");
            when(filePart.getSize()).thenReturn(5L);
            when(filePart.getInputStream()).thenReturn(new ByteArrayInputStream("Hello".getBytes()));

            HttpServletRequest servletRequest = mock(HttpServletRequest.class);
            Collection<Part> parts = new ArrayList<>(List.of(filePart));
            when(servletRequest.getParts()).thenReturn(parts);

            MultipartHttpServletRequest multipartRequest = new MultipartHttpServletRequest(servletRequest);
            Object result = resolver.resolveArgument(param, multipartRequest);

            assertThat(result).isInstanceOf(MultipartFile.class);
            assertThat(((MultipartFile) result).getOriginalFilename()).isEqualTo("test.txt");
        }

        @Test
        @DisplayName("should throw when request is not multipart")
        void shouldThrow_WhenNotMultipartRequest() throws Exception {
            Method method = MultipartFileArgumentResolverTest.class
                    .getDeclaredMethod("uploadFile", MultipartFile.class);
            MethodParameter param = new MethodParameter(method, 0);
            HttpServletRequest plainRequest = mock(HttpServletRequest.class);

            assertThatThrownBy(() -> resolver.resolveArgument(param, plainRequest))
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("not a multipart request");
        }
    }
}
```

#### File: `src/test/java/com/simplespringmvc/integration/MultipartFileUploadIntegrationTest.java` [NEW]

```java
package com.simplespringmvc.integration;

import com.simplespringmvc.annotation.*;
import com.simplespringmvc.container.SimpleBeanContainer;
import com.simplespringmvc.multipart.MultipartFile;
import com.simplespringmvc.multipart.StandardServletMultipartResolver;
import com.simplespringmvc.server.EmbeddedTomcat;
import com.simplespringmvc.servlet.SimpleDispatcherServlet;
import org.junit.jupiter.api.*;

import java.io.IOException;
import java.net.URI;
import java.net.http.*;
import java.nio.charset.StandardCharsets;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class MultipartFileUploadIntegrationTest {

    @Controller
    static class UploadController {

        @PostMapping("/upload")
        @ResponseBody
        public String handleUpload(@RequestParam("file") MultipartFile file) throws IOException {
            return "name=" + file.getOriginalFilename()
                    + ",type=" + file.getContentType()
                    + ",size=" + file.getSize()
                    + ",content=" + new String(file.getBytes(), StandardCharsets.UTF_8);
        }

        @PostMapping("/upload-with-param")
        @ResponseBody
        public String handleUploadWithParam(@RequestParam("file") MultipartFile file,
                                             @RequestParam("description") String description) {
            return "file=" + file.getOriginalFilename() + ",desc=" + description;
        }
    }

    private EmbeddedTomcat tomcat;
    private HttpClient httpClient;
    private String baseUrl;

    @BeforeEach
    void setUp() throws Exception {
        SimpleBeanContainer container = new SimpleBeanContainer();
        container.registerBean(new UploadController());
        SimpleDispatcherServlet servlet = new SimpleDispatcherServlet(container);
        servlet.setMultipartResolver(new StandardServletMultipartResolver());
        servlet.init();
        tomcat = new EmbeddedTomcat(0, servlet);
        tomcat.enableMultipart();
        tomcat.start();
        baseUrl = "http://localhost:" + tomcat.getPort();
        httpClient = HttpClient.newHttpClient();
    }

    @AfterEach
    void tearDown() throws Exception {
        if (tomcat != null) tomcat.stop();
    }

    @Test
    @DisplayName("should upload file and receive metadata in response")
    void shouldUploadFile_WhenSingleFilePosted() throws Exception {
        String boundary = UUID.randomUUID().toString();
        String body = "--" + boundary + "\r\n"
                + "Content-Disposition: form-data; name=\"file\"; filename=\"hello.txt\"\r\n"
                + "Content-Type: text/plain\r\n\r\n"
                + "Hello, World!\r\n"
                + "--" + boundary + "--\r\n";

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder()
                        .uri(URI.create(baseUrl + "/upload"))
                        .header("Content-Type", "multipart/form-data; boundary=" + boundary)
                        .POST(HttpRequest.BodyPublishers.ofString(body))
                        .build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).contains("name=hello.txt");
        assertThat(response.body()).contains("content=Hello, World!");
    }

    @Test
    @DisplayName("should upload file with additional form parameter")
    void shouldUploadFile_WhenFileAndParamPosted() throws Exception {
        String boundary = UUID.randomUUID().toString();
        String body = "--" + boundary + "\r\n"
                + "Content-Disposition: form-data; name=\"file\"; filename=\"doc.pdf\"\r\n"
                + "Content-Type: application/pdf\r\n\r\n"
                + "fake pdf\r\n"
                + "--" + boundary + "\r\n"
                + "Content-Disposition: form-data; name=\"description\"\r\n\r\n"
                + "My document\r\n"
                + "--" + boundary + "--\r\n";

        HttpResponse<String> response = httpClient.send(
                HttpRequest.newBuilder()
                        .uri(URI.create(baseUrl + "/upload-with-param"))
                        .header("Content-Type", "multipart/form-data; boundary=" + boundary)
                        .POST(HttpRequest.BodyPublishers.ofString(body))
                        .build(),
                HttpResponse.BodyHandlers.ofString());

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).contains("file=doc.pdf");
        assertThat(response.body()).contains("desc=My document");
    }
}
```

---

## Summary

| Concept | What It Means |
|---------|---------------|
| **MultipartFile** | Interface abstracting uploaded files — decouples controllers from the Servlet Part API |
| **MultipartResolver** | Strategy that detects multipart requests and wraps them for file access |
| **MultipartHttpServletRequest** | Decorator wrapping the original request, parsing Parts into MultipartFiles |
| **StandardMultipartFile** | Adapter bridging Servlet 3.0 `Part` to Spring's `MultipartFile` |
| **MultipartConfigElement** | Servlet container configuration enabling `getParts()` — required at the Tomcat level |
| **Two-layer config** | Both the Servlet container (MultipartConfigElement) and framework (MultipartResolver) must be configured |

**Next: Chapter 26 — XML Message Conversion** — Extend the `HttpMessageConverter` pattern to serialize/deserialize XML via JAXB and Jackson XML, demonstrating how the converter list from ch06 and content negotiation from ch14 make adding new formats trivial.
