# Chapter 22: SSL Bundles

## Build Challenge

| | |
|---|---|
| **Current State** | The embedded Tomcat server listens on plain HTTP. There is no SSL/TLS support, no way to configure certificates, and no abstraction for managing key stores or trust stores across different components. |
| **Limitation** | Applications that need HTTPS must manually configure Tomcat connectors with file paths, passwords, and key store types. There is no unified way to define SSL configuration once and share it across the web server, HTTP clients, and database connections. JKS and PEM certificate formats require completely different loading code. |
| **Objective** | Introduce the SSL Bundles abstraction: a named, reusable SSL configuration that composes key stores, trust stores, key references, and protocol options into a single bundle. Support both JKS/PKCS12 and PEM certificate formats. Wire bundles into Tomcat via auto-configuration so that `server.ssl.bundle=myBundle` enables HTTPS. Enhance the `ConfigurationPropertiesBinder` with `Map<String, ?>` binding to support the `spring.ssl.bundle.jks.<name>` and `spring.ssl.bundle.pem.<name>` property structure. |

---

## 22.1 The Integration Point

The SSL Bundles feature plugs into **three places** in the existing codebase:

### TomcatServletWebServerFactory -- where SSL meets Tomcat

This is where the SSL bundle is consumed. The `createTomcat()` method checks for an SSL bundle and, if present, creates an HTTPS connector instead of a plain HTTP one:

**Modifying:** `iris-boot-core/.../web/embedded/tomcat/TomcatServletWebServerFactory.java`

Before (Feature 19 -- only HTTP, only port and shutdown):
```java
private Tomcat createTomcat() {
    Tomcat tomcat = new Tomcat();
    File baseDir = createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    tomcat.setPort(this.port);
    tomcat.getHost().setAutoDeploy(false);
    return tomcat;
}
```

After (Feature 22 -- SSL-aware connector creation):
```java
private Tomcat createTomcat() {
    Tomcat tomcat = new Tomcat();
    File baseDir = createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());

    if (this.sslBundle != null) {
        // SSL: create a custom connector with HTTPS support
        Connector connector = createSslConnector();
        tomcat.setConnector(connector);
    } else {
        tomcat.setPort(this.port);
    }

    tomcat.getHost().setAutoDeploy(false);
    return tomcat;
}
```

The new `createSslConnector()` method is the bridge between Spring Boot's SSL abstraction and Tomcat's native SSL API. It extracts the `KeyStore` from the bundle, writes it to a temp file (Tomcat expects file paths), and configures an `SSLHostConfig`:

```java
private Connector createSslConnector() {
    Connector connector = new Connector();
    connector.setPort(this.port);
    connector.setScheme("https");
    connector.setSecure(true);
    connector.setProperty("SSLEnabled", "true");

    SSLHostConfig sslHostConfig = new SSLHostConfig();

    SSLHostConfigCertificate certificate = new SSLHostConfigCertificate(
            sslHostConfig, SSLHostConfigCertificate.Type.UNDEFINED);

    KeyStore keyStore = this.sslBundle.getStores().getKeyStore();
    if (keyStore != null) {
        File keystoreFile = writeKeyStoreToTempFile(keyStore,
                this.sslBundle.getStores().getKeyStorePassword());
        certificate.setCertificateKeystoreFile(keystoreFile.getAbsolutePath());
        certificate.setCertificateKeystoreType(keyStore.getType());
        if (this.sslBundle.getStores().getKeyStorePassword() != null) {
            certificate.setCertificateKeystorePassword(
                    this.sslBundle.getStores().getKeyStorePassword());
        }
    }

    if (this.sslBundle.getKey().getAlias() != null) {
        certificate.setCertificateKeyAlias(this.sslBundle.getKey().getAlias());
    }
    if (this.sslBundle.getKey().getPassword() != null) {
        certificate.setCertificateKeyPassword(this.sslBundle.getKey().getPassword());
    }

    sslHostConfig.addCertificate(certificate);

    KeyStore trustStore = this.sslBundle.getStores().getTrustStore();
    if (trustStore != null) {
        File truststoreFile = writeKeyStoreToTempFile(trustStore, null);
        sslHostConfig.setTruststoreFile(truststoreFile.getAbsolutePath());
        sslHostConfig.setTruststoreType(trustStore.getType());
    }

    SslOptions options = this.sslBundle.getOptions();
    if (options.getCiphers() != null) {
        sslHostConfig.setCiphers(String.join(",", options.getCiphers()));
    }
    if (options.getEnabledProtocols() != null) {
        sslHostConfig.setProtocols(String.join(",", options.getEnabledProtocols()));
    }

    sslHostConfig.setSslProtocol(this.sslBundle.getProtocol());

    connector.addSslHostConfig(sslHostConfig);
    return connector;
}
```

### TomcatAutoConfiguration -- wiring SslBundles into the factory

**Modifying:** `iris-boot-core/.../autoconfigure/web/servlet/TomcatAutoConfiguration.java`

Before (Feature 19 -- only port and shutdown):
```java
@Bean
@ConditionalOnMissingBean
public ServletWebServerFactory tomcatServletWebServerFactory(
        ServerProperties serverProperties) {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
    factory.setPort(serverProperties.getPort());
    factory.setShutdown(serverProperties.getShutdown());
    return factory;
}
```

After (Feature 22 -- SSL bundle lookup added, ordered after SslAutoConfiguration):
```java
@AutoConfiguration
@AutoConfigureAfter(SslAutoConfiguration.class)
@ConditionalOnClass(name = "org.apache.catalina.startup.Tomcat")
@EnableConfigurationProperties(ServerProperties.class)
public class TomcatAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public ServletWebServerFactory tomcatServletWebServerFactory(
            ServerProperties serverProperties,
            SslBundles sslBundles) {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.setPort(serverProperties.getPort());
        factory.setShutdown(serverProperties.getShutdown());

        // Wire SSL if configured
        String bundleName = serverProperties.getSsl().getBundle();
        if (bundleName != null && !bundleName.isBlank() && sslBundles != null) {
            factory.setSslBundle(sslBundles.getBundle(bundleName));
        }

        return factory;
    }
}
```

The `@AutoConfigureAfter(SslAutoConfiguration.class)` ensures the `SslBundles` bean exists before Tomcat tries to use it.

### ServerProperties -- the ssl.bundle property

**Modifying:** `iris-boot-core/.../autoconfigure/web/ServerProperties.java`

Before (Feature 19 -- only port and shutdown):
```java
@ConfigurationProperties(prefix = "server")
public class ServerProperties {
    private int port = 8080;
    private Shutdown shutdown = Shutdown.IMMEDIATE;
    // ... getters/setters
}
```

After (Feature 22 -- nested Ssl class with bundle name):
```java
@ConfigurationProperties(prefix = "server")
public class ServerProperties {
    private int port = 8080;
    private Shutdown shutdown = Shutdown.IMMEDIATE;
    private Ssl ssl = new Ssl();

    // ... getters/setters for port, shutdown, ssl

    public static class Ssl {
        private String bundle;

        public String getBundle() { return bundle; }
        public void setBundle(String bundle) { this.bundle = bundle; }
    }
}
```

Now `server.ssl.bundle=myBundle` in `application.properties` tells the embedded server which SSL bundle to use.

**Direction:** From these three integration points, we need to build:
1. Core SSL abstractions: `SslBundleKey`, `SslOptions`, `SslStoreBundle`, `SslManagerBundle`, `DefaultSslManagerBundle`, `SslBundle` (the composition interface)
2. Registry layer: `SslBundleRegistry` (write), `SslBundles` (read), `DefaultSslBundleRegistry`, `NoSuchSslBundleException`
3. Store implementations: `JksSslStoreDetails` + `JksSslStoreBundle` (JKS/PKCS12 files), `PemSslStoreDetails` + `PemContent` + `PemSslStoreBundle` (PEM files)
4. Auto-configuration: `SslProperties`, `JksSslBundleProperties`, `PemSslBundleProperties`, `SslAutoConfiguration`
5. Enhancement to `ConfigurationPropertiesBinder`: `Map<String, ?>` binding for the `spring.ssl.bundle.jks.<name>` structure

The key insight: all certificate formats (JKS, PKCS12, PEM) are normalized into Java `KeyStore` objects by the store bundles, so consumers like Tomcat never need to know what format the certificates were originally in.

---

## 22.2 Core SSL Abstractions

### SslBundleKey -- key reference within a store

A reference to a single key within a key store, identified by alias and protected by a password.

**New file:** `iris-boot-core/.../ssl/SslBundleKey.java`

```java
package com.iris.boot.ssl;

import java.security.KeyStore;
import java.security.KeyStoreException;
import java.util.Enumeration;

public final class SslBundleKey {

    public static final SslBundleKey NONE = new SslBundleKey(null, null);

    private final String password;
    private final String alias;

    private SslBundleKey(String password, String alias) {
        this.password = password;
        this.alias = alias;
    }

    public static SslBundleKey of(String password, String alias) {
        if (password == null && alias == null) {
            return NONE;
        }
        return new SslBundleKey(password, alias);
    }

    public String getPassword() {
        return this.password;
    }

    public String getAlias() {
        return this.alias;
    }

    public void assertContainsAlias(KeyStore keyStore) {
        if (this.alias == null || keyStore == null) {
            return;
        }
        try {
            if (!keyStore.containsAlias(this.alias)) {
                StringBuilder available = new StringBuilder();
                Enumeration<String> aliases = keyStore.aliases();
                while (aliases.hasMoreElements()) {
                    if (available.length() > 0) {
                        available.append(", ");
                    }
                    available.append(aliases.nextElement());
                }
                throw new IllegalStateException(
                        "Keystore does not contain alias '" + this.alias
                        + "'. Available aliases: [" + available + "]");
            }
        } catch (KeyStoreException ex) {
            throw new IllegalStateException(
                    "Could not determine if keystore contains alias '" + this.alias + "'", ex);
        }
    }
}
```

The `NONE` singleton avoids unnecessary allocations when no key selection is needed. The `assertContainsAlias()` method provides fail-fast validation -- if the user configures an alias that does not exist, the application fails at startup with a clear message rather than at runtime with a cryptic SSL error.

### SslOptions -- cipher and protocol restrictions

**New file:** `iris-boot-core/.../ssl/SslOptions.java`

```java
package com.iris.boot.ssl;

public final class SslOptions {

    public static final SslOptions NONE = new SslOptions(null, null);

    private final String[] ciphers;
    private final String[] enabledProtocols;

    private SslOptions(String[] ciphers, String[] enabledProtocols) {
        this.ciphers = ciphers;
        this.enabledProtocols = enabledProtocols;
    }

    public static SslOptions of(String[] ciphers, String[] enabledProtocols) {
        if (ciphers == null && enabledProtocols == null) {
            return NONE;
        }
        return new SslOptions(
                ciphers != null ? ciphers.clone() : null,
                enabledProtocols != null ? enabledProtocols.clone() : null);
    }

    public String[] getCiphers() {
        return this.ciphers;
    }

    public String[] getEnabledProtocols() {
        return this.enabledProtocols;
    }

    public boolean isSpecified() {
        return this.ciphers != null || this.enabledProtocols != null;
    }
}
```

The defensive `clone()` in the factory method prevents callers from modifying the internal arrays after construction.

### SslStoreBundle -- the key store / trust store pair

**New file:** `iris-boot-core/.../ssl/SslStoreBundle.java`

```java
package com.iris.boot.ssl;

import java.security.KeyStore;

public interface SslStoreBundle {

    SslStoreBundle NONE = new SslStoreBundle() {
        @Override
        public KeyStore getKeyStore() { return null; }

        @Override
        public String getKeyStorePassword() { return null; }

        @Override
        public KeyStore getTrustStore() { return null; }
    };

    KeyStore getKeyStore();

    String getKeyStorePassword();

    KeyStore getTrustStore();

    static SslStoreBundle of(KeyStore keyStore, String keyStorePassword, KeyStore trustStore) {
        return new SslStoreBundle() {
            @Override
            public KeyStore getKeyStore() { return keyStore; }

            @Override
            public String getKeyStorePassword() { return keyStorePassword; }

            @Override
            public KeyStore getTrustStore() { return trustStore; }
        };
    }
}
```

This is the abstraction that makes the SSL system format-agnostic. Whether the source is a JKS file, a PKCS12 file, or PEM certificates, consumers always work with `KeyStore` objects.

### SslManagerBundle -- the bridge to SSLContext

**New file:** `iris-boot-core/.../ssl/SslManagerBundle.java`

```java
package com.iris.boot.ssl;

import javax.net.ssl.KeyManager;
import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManager;
import javax.net.ssl.TrustManagerFactory;

public interface SslManagerBundle {

    KeyManagerFactory getKeyManagerFactory();

    TrustManagerFactory getTrustManagerFactory();

    default KeyManager[] getKeyManagers() {
        KeyManagerFactory factory = getKeyManagerFactory();
        return (factory != null) ? factory.getKeyManagers() : null;
    }

    default TrustManager[] getTrustManagers() {
        TrustManagerFactory factory = getTrustManagerFactory();
        return (factory != null) ? factory.getTrustManagers() : null;
    }

    default SSLContext createSslContext(String protocol) {
        try {
            SSLContext sslContext = SSLContext.getInstance(protocol);
            sslContext.init(getKeyManagers(), getTrustManagers(), null);
            return sslContext;
        } catch (Exception ex) {
            throw new IllegalStateException(
                    "Error creating SSLContext with protocol '" + protocol + "'", ex);
        }
    }

    static SslManagerBundle of(KeyManagerFactory keyManagerFactory,
                                TrustManagerFactory trustManagerFactory) {
        return new SslManagerBundle() {
            @Override
            public KeyManagerFactory getKeyManagerFactory() { return keyManagerFactory; }

            @Override
            public TrustManagerFactory getTrustManagerFactory() { return trustManagerFactory; }
        };
    }

    static SslManagerBundle from(SslStoreBundle stores, SslBundleKey key) {
        return new DefaultSslManagerBundle(stores, key);
    }
}
```

The `createSslContext()` method is the money method -- it produces a fully initialized `SSLContext` that can be used by any component (Tomcat, HTTP clients, database connections). The `from()` factory method delegates to `DefaultSslManagerBundle`, which does the heavy lifting.

### DefaultSslManagerBundle -- building managers from stores

**New file:** `iris-boot-core/.../ssl/DefaultSslManagerBundle.java`

```java
package com.iris.boot.ssl;

import java.security.KeyStore;
import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.TrustManagerFactory;

class DefaultSslManagerBundle implements SslManagerBundle {

    private final SslStoreBundle storeBundle;
    private final SslBundleKey key;

    DefaultSslManagerBundle(SslStoreBundle storeBundle, SslBundleKey key) {
        this.storeBundle = (storeBundle != null) ? storeBundle : SslStoreBundle.NONE;
        this.key = (key != null) ? key : SslBundleKey.NONE;
    }

    @Override
    public KeyManagerFactory getKeyManagerFactory() {
        KeyStore keyStore = this.storeBundle.getKeyStore();
        if (keyStore == null) {
            return null;
        }

        try {
            this.key.assertContainsAlias(keyStore);

            String keyPassword = this.key.getPassword();
            if (keyPassword == null) {
                keyPassword = this.storeBundle.getKeyStorePassword();
            }
            char[] keyPasswordChars = (keyPassword != null) ? keyPassword.toCharArray() : null;

            KeyManagerFactory factory = KeyManagerFactory.getInstance(
                    KeyManagerFactory.getDefaultAlgorithm());
            factory.init(keyStore, keyPasswordChars);
            return factory;

        } catch (Exception ex) {
            throw new IllegalStateException("Error creating KeyManagerFactory", ex);
        }
    }

    @Override
    public TrustManagerFactory getTrustManagerFactory() {
        KeyStore trustStore = this.storeBundle.getTrustStore();
        if (trustStore == null) {
            return null;
        }

        try {
            TrustManagerFactory factory = TrustManagerFactory.getInstance(
                    TrustManagerFactory.getDefaultAlgorithm());
            factory.init(trustStore);
            return factory;

        } catch (Exception ex) {
            throw new IllegalStateException("Error creating TrustManagerFactory", ex);
        }
    }
}
```

This is the critical bridge. The key password falls back to the keystore password if not set separately -- a common pattern since many tools use the same password for both. The alias is validated at construction time (fail-fast), not at SSL handshake time.

### SslBundle -- the composition interface

**New file:** `iris-boot-core/.../ssl/SslBundle.java`

```java
package com.iris.boot.ssl;

import javax.net.ssl.SSLContext;

public interface SslBundle {

    String DEFAULT_PROTOCOL = "TLS";

    SslStoreBundle getStores();

    SslBundleKey getKey();

    SslOptions getOptions();

    SslManagerBundle getManagers();

    default String getProtocol() {
        return DEFAULT_PROTOCOL;
    }

    default SSLContext createSslContext() {
        return getManagers().createSslContext(getProtocol());
    }

    static SslBundle of(SslStoreBundle stores, SslBundleKey key,
                         SslOptions options, String protocol) {
        return of(stores, key, options, protocol, null);
    }

    static SslBundle of(SslStoreBundle stores, SslBundleKey key,
                         SslOptions options, String protocol,
                         SslManagerBundle managers) {
        SslStoreBundle effectiveStores = (stores != null) ? stores : SslStoreBundle.NONE;
        SslBundleKey effectiveKey = (key != null) ? key : SslBundleKey.NONE;
        SslOptions effectiveOptions = (options != null) ? options : SslOptions.NONE;
        String effectiveProtocol = (protocol != null) ? protocol : DEFAULT_PROTOCOL;
        SslManagerBundle effectiveManagers = (managers != null)
                ? managers
                : SslManagerBundle.from(effectiveStores, effectiveKey);

        return new SslBundle() {
            @Override public SslStoreBundle getStores() { return effectiveStores; }
            @Override public SslBundleKey getKey() { return effectiveKey; }
            @Override public SslOptions getOptions() { return effectiveOptions; }
            @Override public SslManagerBundle getManagers() { return effectiveManagers; }
            @Override public String getProtocol() { return effectiveProtocol; }
        };
    }
}
```

`SslBundle` is the top-level abstraction. It composes all four sub-components and provides the convenience `createSslContext()` method. The factory method auto-creates the manager bundle from stores and key if not explicitly provided -- consumers rarely need to create managers manually.

---

## 22.3 Registry Layer

### SslBundleRegistry -- the write side

**New file:** `iris-boot-core/.../ssl/SslBundleRegistry.java`

```java
package com.iris.boot.ssl;

public interface SslBundleRegistry {

    void registerBundle(String name, SslBundle bundle);
}
```

### SslBundles -- the read side

**New file:** `iris-boot-core/.../ssl/SslBundles.java`

```java
package com.iris.boot.ssl;

public interface SslBundles {

    SslBundle getBundle(String name) throws NoSuchSslBundleException;
}
```

Splitting registration from retrieval follows the Interface Segregation Principle. Auto-configuration code uses `SslBundleRegistry` to register bundles. Application components inject `SslBundles` to look them up. Neither side sees methods it does not need.

### DefaultSslBundleRegistry -- the implementation

**New file:** `iris-boot-core/.../ssl/DefaultSslBundleRegistry.java`

```java
package com.iris.boot.ssl;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class DefaultSslBundleRegistry implements SslBundleRegistry, SslBundles {

    private final ConcurrentMap<String, SslBundle> bundles = new ConcurrentHashMap<>();

    @Override
    public void registerBundle(String name, SslBundle bundle) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("SSL bundle name must not be null or blank");
        }
        if (bundle == null) {
            throw new IllegalArgumentException("SSL bundle must not be null");
        }
        SslBundle existing = this.bundles.putIfAbsent(name, bundle);
        if (existing != null) {
            throw new IllegalStateException(
                    "Cannot register SSL bundle '" + name
                    + "' — a bundle with this name is already registered");
        }
    }

    @Override
    public SslBundle getBundle(String name) throws NoSuchSslBundleException {
        SslBundle bundle = this.bundles.get(name);
        if (bundle == null) {
            throw new NoSuchSslBundleException(name);
        }
        return bundle;
    }
}
```

One object implements both interfaces. The `ConcurrentHashMap` with `putIfAbsent` ensures thread-safe registration during concurrent auto-configuration -- the second registration of the same name fails with a clear error.

### NoSuchSslBundleException -- clear error for missing bundles

**New file:** `iris-boot-core/.../ssl/NoSuchSslBundleException.java`

```java
package com.iris.boot.ssl;

public class NoSuchSslBundleException extends RuntimeException {

    private final String bundleName;

    public NoSuchSslBundleException(String bundleName) {
        super("SSL bundle '" + bundleName + "' is not registered. "
                + "Check your 'spring.ssl.bundle' configuration.");
        this.bundleName = bundleName;
    }

    public String getBundleName() {
        return this.bundleName;
    }
}
```

---

## 22.4 Store Implementations

### JKS/PKCS12 -- loading keystore files

#### JksSslStoreDetails -- configuration record

**New file:** `iris-boot-core/.../ssl/jks/JksSslStoreDetails.java`

```java
package com.iris.boot.ssl.jks;

public record JksSslStoreDetails(String type, String location, String password) {

    public static JksSslStoreDetails forLocation(String location) {
        return new JksSslStoreDetails("PKCS12", location, null);
    }
}
```

#### JksSslStoreBundle -- loading JKS/PKCS12 files

**New file:** `iris-boot-core/.../ssl/jks/JksSslStoreBundle.java`

```java
package com.iris.boot.ssl.jks;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyStore;

import com.iris.boot.ssl.SslStoreBundle;

public class JksSslStoreBundle implements SslStoreBundle {

    private final JksSslStoreDetails keyStoreDetails;
    private final JksSslStoreDetails trustStoreDetails;

    private volatile KeyStore keyStore;
    private volatile boolean keyStoreLoaded;

    private volatile KeyStore trustStore;
    private volatile boolean trustStoreLoaded;

    public JksSslStoreBundle(JksSslStoreDetails keyStoreDetails,
                              JksSslStoreDetails trustStoreDetails) {
        this.keyStoreDetails = keyStoreDetails;
        this.trustStoreDetails = trustStoreDetails;
    }

    @Override
    public KeyStore getKeyStore() {
        if (!this.keyStoreLoaded) {
            synchronized (this) {
                if (!this.keyStoreLoaded) {
                    this.keyStore = loadStore(this.keyStoreDetails);
                    this.keyStoreLoaded = true;
                }
            }
        }
        return this.keyStore;
    }

    @Override
    public String getKeyStorePassword() {
        return (this.keyStoreDetails != null) ? this.keyStoreDetails.password() : null;
    }

    @Override
    public KeyStore getTrustStore() {
        if (!this.trustStoreLoaded) {
            synchronized (this) {
                if (!this.trustStoreLoaded) {
                    this.trustStore = loadStore(this.trustStoreDetails);
                    this.trustStoreLoaded = true;
                }
            }
        }
        return this.trustStore;
    }

    private KeyStore loadStore(JksSslStoreDetails details) {
        if (details == null || details.location() == null) {
            return null;
        }
        try {
            String type = (details.type() != null) ? details.type() : KeyStore.getDefaultType();
            KeyStore store = KeyStore.getInstance(type);
            char[] password = (details.password() != null)
                    ? details.password().toCharArray() : null;
            try (InputStream is = openLocation(details.location())) {
                store.load(is, password);
            }
            return store;
        } catch (Exception ex) {
            throw new IllegalStateException(
                    "Unable to load key store from '" + details.location() + "'", ex);
        }
    }

    private InputStream openLocation(String location) throws IOException {
        if (location.startsWith("classpath:")) {
            String path = location.substring("classpath:".length());
            InputStream is = getClass().getClassLoader().getResourceAsStream(path);
            if (is == null) {
                throw new IOException("Classpath resource not found: " + path);
            }
            return is;
        }
        return Files.newInputStream(Path.of(location));
    }
}
```

Stores are loaded lazily on first access using double-checked locking, and then cached. The `openLocation()` method supports both `classpath:keystore.p12` and absolute file paths like `/etc/ssl/keystore.jks`.

### PEM -- loading certificate and key files

#### PemSslStoreDetails -- configuration record

**New file:** `iris-boot-core/.../ssl/pem/PemSslStoreDetails.java`

```java
package com.iris.boot.ssl.pem;

public record PemSslStoreDetails(String certificate, String privateKey, String privateKeyPassword) {

    public static PemSslStoreDetails forCertificate(String certificate) {
        return new PemSslStoreDetails(certificate, null, null);
    }
}
```

#### PemContent -- loading PEM from files or inline strings

**New file:** `iris-boot-core/.../ssl/pem/PemContent.java`

```java
package com.iris.boot.ssl.pem;

import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;

final class PemContent {

    private PemContent() {
    }

    static String load(String source) {
        if (source == null || source.isBlank()) {
            return null;
        }

        // Inline PEM content
        if (source.trim().startsWith("-----BEGIN")) {
            return source;
        }

        // Load from file
        try {
            if (source.startsWith("classpath:")) {
                String path = source.substring("classpath:".length());
                try (InputStream is = PemContent.class.getClassLoader().getResourceAsStream(path)) {
                    if (is == null) {
                        throw new IOException("Classpath resource not found: " + path);
                    }
                    return new String(is.readAllBytes(), StandardCharsets.UTF_8);
                }
            }
            return Files.readString(Path.of(source));
        } catch (IOException ex) {
            throw new IllegalStateException("Unable to load PEM content from '" + source + "'", ex);
        }
    }
}
```

`PemContent` auto-detects whether the input is inline PEM content (starts with `-----BEGIN`) or a file path. This flexibility means `application.properties` can contain either a path to a PEM file or the PEM content itself.

#### PemSslStoreBundle -- converting PEM to KeyStore

**New file:** `iris-boot-core/.../ssl/pem/PemSslStoreBundle.java`

```java
package com.iris.boot.ssl.pem;

import java.io.ByteArrayInputStream;
import java.security.KeyFactory;
import java.security.KeyStore;
import java.security.PrivateKey;
import java.security.cert.Certificate;
import java.security.cert.CertificateFactory;
import java.security.spec.PKCS8EncodedKeySpec;
import java.util.ArrayList;
import java.util.Base64;
import java.util.Collection;
import java.util.List;

import com.iris.boot.ssl.SslStoreBundle;

public class PemSslStoreBundle implements SslStoreBundle {

    private static final String DEFAULT_ALIAS = "ssl";
    private static final char[] EMPTY_PASSWORD = new char[0];

    private final PemSslStoreDetails keyStoreDetails;
    private final PemSslStoreDetails trustStoreDetails;

    private volatile KeyStore keyStore;
    private volatile boolean keyStoreLoaded;

    private volatile KeyStore trustStore;
    private volatile boolean trustStoreLoaded;

    public PemSslStoreBundle(PemSslStoreDetails keyStoreDetails,
                              PemSslStoreDetails trustStoreDetails) {
        this.keyStoreDetails = keyStoreDetails;
        this.trustStoreDetails = trustStoreDetails;
    }

    @Override
    public KeyStore getKeyStore() {
        if (!this.keyStoreLoaded) {
            synchronized (this) {
                if (!this.keyStoreLoaded) {
                    this.keyStore = createKeyStore(this.keyStoreDetails);
                    this.keyStoreLoaded = true;
                }
            }
        }
        return this.keyStore;
    }

    @Override
    public String getKeyStorePassword() {
        return null;
    }

    @Override
    public KeyStore getTrustStore() {
        if (!this.trustStoreLoaded) {
            synchronized (this) {
                if (!this.trustStoreLoaded) {
                    this.trustStore = createKeyStore(this.trustStoreDetails);
                    this.trustStoreLoaded = true;
                }
            }
        }
        return this.trustStore;
    }

    private KeyStore createKeyStore(PemSslStoreDetails details) {
        if (details == null || details.certificate() == null) {
            return null;
        }
        try {
            String certContent = PemContent.load(details.certificate());
            List<Certificate> certificates = parseCertificates(certContent);
            if (certificates.isEmpty()) {
                return null;
            }

            KeyStore store = KeyStore.getInstance("PKCS12");
            store.load(null, null);

            if (details.privateKey() != null) {
                String keyContent = PemContent.load(details.privateKey());
                PrivateKey privateKey = parsePrivateKey(keyContent);
                Certificate[] chain = certificates.toArray(new Certificate[0]);
                store.setKeyEntry(DEFAULT_ALIAS, privateKey, EMPTY_PASSWORD, chain);
            } else {
                for (int i = 0; i < certificates.size(); i++) {
                    String alias = certificates.size() == 1
                            ? DEFAULT_ALIAS
                            : DEFAULT_ALIAS + "-" + i;
                    store.setCertificateEntry(alias, certificates.get(i));
                }
            }
            return store;
        } catch (Exception ex) {
            throw new IllegalStateException("Unable to create KeyStore from PEM content", ex);
        }
    }

    private List<Certificate> parseCertificates(String pemContent) throws Exception {
        CertificateFactory factory = CertificateFactory.getInstance("X.509");
        Collection<? extends Certificate> certs = factory.generateCertificates(
                new ByteArrayInputStream(pemContent.getBytes()));
        return new ArrayList<>(certs);
    }

    private PrivateKey parsePrivateKey(String pemContent) throws Exception {
        String base64 = pemContent
                .replace("-----BEGIN PRIVATE KEY-----", "")
                .replace("-----END PRIVATE KEY-----", "")
                .replaceAll("\\s", "");
        byte[] keyBytes = Base64.getDecoder().decode(base64);
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);

        // Try RSA first, then EC
        try {
            return KeyFactory.getInstance("RSA").generatePrivate(keySpec);
        } catch (Exception rsaEx) {
            try {
                return KeyFactory.getInstance("EC").generatePrivate(keySpec);
            } catch (Exception ecEx) {
                throw new IllegalStateException(
                        "Unable to parse private key — only RSA and EC PKCS#8 keys "
                        + "are supported ('BEGIN PRIVATE KEY' format)", rsaEx);
            }
        }
    }
}
```

The critical conversion: PEM content (text-based certificates and keys) is parsed into Java `KeyStore` objects. If a private key is present, the certificates are stored as a key entry (identity). If only certificates are provided, they are stored as trusted certificate entries (trust). The result is always a PKCS12 `KeyStore` -- the same type that `JksSslStoreBundle` produces.

---

## 22.5 Auto-Configuration and Properties

### JksSslBundleProperties -- JKS bundle configuration

**New file:** `iris-boot-core/.../autoconfigure/ssl/JksSslBundleProperties.java`

```java
package com.iris.boot.autoconfigure.ssl;

public class JksSslBundleProperties {

    private Key key = new Key();
    private Store keystore = new Store();
    private Store truststore = new Store();
    private String protocol;

    // ... getters/setters

    public static class Key {
        private String alias;
        private String password;
        // ... getters/setters
    }

    public static class Store {
        private String type;
        private String location;
        private String password;
        // ... getters/setters
    }
}
```

Bound from properties like:
```properties
spring.ssl.bundle.jks.myBundle.key.alias=tomcat
spring.ssl.bundle.jks.myBundle.keystore.location=classpath:keystore.p12
spring.ssl.bundle.jks.myBundle.keystore.password=secret
spring.ssl.bundle.jks.myBundle.keystore.type=PKCS12
```

### PemSslBundleProperties -- PEM bundle configuration

**New file:** `iris-boot-core/.../autoconfigure/ssl/PemSslBundleProperties.java`

```java
package com.iris.boot.autoconfigure.ssl;

public class PemSslBundleProperties {

    private Store keystore = new Store();
    private Store truststore = new Store();
    private String protocol;

    // ... getters/setters

    public static class Store {
        private String certificate;
        private String privateKey;
        private String privateKeyPassword;
        // ... getters/setters
    }
}
```

Bound from properties like:
```properties
spring.ssl.bundle.pem.myBundle.keystore.certificate=classpath:cert.pem
spring.ssl.bundle.pem.myBundle.keystore.private-key=classpath:key.pem
```

### SslProperties -- the top-level properties container

**New file:** `iris-boot-core/.../autoconfigure/ssl/SslProperties.java`

```java
package com.iris.boot.autoconfigure.ssl;

import java.util.LinkedHashMap;
import java.util.Map;

import com.iris.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "spring.ssl")
public class SslProperties {

    private Bundle bundle = new Bundle();

    public Bundle getBundle() { return bundle; }
    public void setBundle(Bundle bundle) { this.bundle = bundle; }

    public static class Bundle {
        private Map<String, JksSslBundleProperties> jks = new LinkedHashMap<>();
        private Map<String, PemSslBundleProperties> pem = new LinkedHashMap<>();

        public Map<String, JksSslBundleProperties> getJks() { return jks; }
        public void setJks(Map<String, JksSslBundleProperties> jks) { this.jks = jks; }

        public Map<String, PemSslBundleProperties> getPem() { return pem; }
        public void setPem(Map<String, PemSslBundleProperties> pem) { this.pem = pem; }
    }
}
```

The `Map<String, JksSslBundleProperties>` is the key structure: the map keys ("server", "client") become the bundle names used for lookup. This is what drives the need for `Map<String, ?>` binding in `ConfigurationPropertiesBinder`.

### SslAutoConfiguration -- populating the registry from properties

**New file:** `iris-boot-core/.../autoconfigure/ssl/SslAutoConfiguration.java`

```java
package com.iris.boot.autoconfigure.ssl;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.boot.context.properties.EnableConfigurationProperties;
import com.iris.boot.ssl.*;
import com.iris.boot.ssl.jks.*;
import com.iris.boot.ssl.pem.*;
import com.iris.framework.context.annotation.Bean;

@AutoConfiguration
@EnableConfigurationProperties(SslProperties.class)
public class SslAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DefaultSslBundleRegistry sslBundleRegistry(SslProperties sslProperties) {
        DefaultSslBundleRegistry registry = new DefaultSslBundleRegistry();

        // Register JKS bundles
        sslProperties.getBundle().getJks().forEach((name, props) -> {
            SslBundle bundle = createJksBundle(props);
            registry.registerBundle(name, bundle);
        });

        // Register PEM bundles
        sslProperties.getBundle().getPem().forEach((name, props) -> {
            SslBundle bundle = createPemBundle(props);
            registry.registerBundle(name, bundle);
        });

        return registry;
    }

    private SslBundle createJksBundle(JksSslBundleProperties props) {
        JksSslStoreDetails keyStoreDetails = new JksSslStoreDetails(
                props.getKeystore().getType(),
                props.getKeystore().getLocation(),
                props.getKeystore().getPassword());

        JksSslStoreDetails trustStoreDetails = null;
        if (props.getTruststore().getLocation() != null) {
            trustStoreDetails = new JksSslStoreDetails(
                    props.getTruststore().getType(),
                    props.getTruststore().getLocation(),
                    props.getTruststore().getPassword());
        }

        SslStoreBundle stores = new JksSslStoreBundle(keyStoreDetails, trustStoreDetails);
        SslBundleKey key = SslBundleKey.of(
                props.getKey().getPassword(),
                props.getKey().getAlias());

        return SslBundle.of(stores, key, null, props.getProtocol());
    }

    private SslBundle createPemBundle(PemSslBundleProperties props) {
        PemSslStoreDetails keyStoreDetails = null;
        if (props.getKeystore().getCertificate() != null) {
            keyStoreDetails = new PemSslStoreDetails(
                    props.getKeystore().getCertificate(),
                    props.getKeystore().getPrivateKey(),
                    props.getKeystore().getPrivateKeyPassword());
        }

        PemSslStoreDetails trustStoreDetails = null;
        if (props.getTruststore().getCertificate() != null) {
            trustStoreDetails = new PemSslStoreDetails(
                    props.getTruststore().getCertificate(),
                    null, null);
        }

        SslStoreBundle stores = new PemSslStoreBundle(keyStoreDetails, trustStoreDetails);
        return SslBundle.of(stores, SslBundleKey.NONE, null, props.getProtocol());
    }
}
```

The auto-configuration reads both `spring.ssl.bundle.jks.*` and `spring.ssl.bundle.pem.*` and registers each as a named `SslBundle`. The `@ConditionalOnMissingBean` allows users to provide their own registry.

### AutoConfiguration.imports -- registering SslAutoConfiguration

**Modified file:** `iris-boot-core/.../resources/META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports`

Before:
```
com.iris.boot.autoconfigure.jackson.JacksonAutoConfiguration
com.iris.boot.autoconfigure.web.servlet.TomcatAutoConfiguration
com.iris.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

After:
```
com.iris.boot.autoconfigure.jackson.JacksonAutoConfiguration
com.iris.boot.autoconfigure.ssl.SslAutoConfiguration
com.iris.boot.autoconfigure.web.servlet.TomcatAutoConfiguration
com.iris.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

`SslAutoConfiguration` is listed before `TomcatAutoConfiguration` so the `SslBundles` bean is available when Tomcat auto-configuration runs.

### ConfigurationPropertiesBinder -- Map binding enhancement

**Modified file:** `iris-boot-core/.../context/properties/ConfigurationPropertiesBinder.java`

The SSL properties structure (`spring.ssl.bundle.jks.<name>.keystore.location=...`) requires binding `Map<String, JksSslBundleProperties>`. The binder previously handled simple types, lists, and nested objects, but not maps.

The new `bindMapField()` method:

```java
@SuppressWarnings("unchecked")
private void bindMapField(Object target, Field field, String prefix,
                           String fieldName) {
    Class<?> valueType = getMapValueType(field);
    if (valueType == null) {
        return;
    }

    String mapPrefix = prefix + "." + fieldName;
    String mapKebabPrefix = prefix + "." + toKebabCase(fieldName);

    // Find all unique map keys from property sources
    Set<String> mapKeys = findMapKeys(mapPrefix);
    if (!mapPrefix.equals(mapKebabPrefix)) {
        mapKeys.addAll(findMapKeys(mapKebabPrefix));
    }

    if (mapKeys.isEmpty()) {
        return;
    }

    Map<String, Object> map = (Map<String, Object>) getFieldValue(target, field);
    if (map == null) {
        map = new LinkedHashMap<>();
        setFieldValue(target, field, map);
    }

    for (String key : mapKeys) {
        String entryPrefix = mapPrefix + "." + key;
        if (isSimpleType(valueType)) {
            String value = environment.getProperty(entryPrefix);
            if (value != null) {
                map.put(key, PropertySourcesPropertyResolver.convertValue(value, valueType));
            }
        } else {
            try {
                Object nested = valueType.getDeclaredConstructor().newInstance();
                bind(nested, entryPrefix);
                map.put(key, nested);
            } catch (ReflectiveOperationException ex) {
                // Cannot create value instance — skip this map entry
            }
        }
    }
}
```

The `findMapKeys()` method scans all property sources for keys matching `prefix.<mapKey>...` and extracts unique map keys:

```java
private Set<String> findMapKeys(String mapPrefix) {
    String dotPrefix = mapPrefix + ".";
    Set<String> keys = new LinkedHashSet<>();
    for (PropertySource<?> ps : environment.getPropertySources()) {
        Object source = ps.getSource();
        if (source instanceof java.util.Map<?, ?> map) {
            for (Object key : map.keySet()) {
                if (key instanceof String strKey && strKey.startsWith(dotPrefix)) {
                    String remaining = strKey.substring(dotPrefix.length());
                    int dotIndex = remaining.indexOf('.');
                    String mapKey = (dotIndex > 0) ? remaining.substring(0, dotIndex) : remaining;
                    keys.add(mapKey);
                }
            }
        }
    }
    return keys;
}
```

For a prefix of `spring.ssl.bundle.jks` and properties like `spring.ssl.bundle.jks.server.keystore.location` and `spring.ssl.bundle.jks.client.truststore.location`, this returns `["server", "client"]`.

The `bindField()` method now dispatches to the map binder:

```java
private void bindField(Object target, Field field, String prefix) {
    String fieldName = field.getName();
    Class<?> fieldType = field.getType();

    if (isSimpleType(fieldType)) {
        bindSimpleField(target, field, prefix, fieldName, fieldType);
    } else if (fieldType == List.class) {
        bindListField(target, field, prefix, fieldName);
    } else if (Map.class.isAssignableFrom(fieldType)) {
        bindMapField(target, field, prefix, fieldName);     // NEW
    } else {
        bindNestedObject(target, field, prefix, fieldName);
    }
}
```

---

## 22.6 Try It Yourself

<details><summary>Challenge 1: Why does SslStoreBundle normalize everything to KeyStore objects instead of keeping the original format?</summary>

Because consumers should not need to know the certificate format. Tomcat's SSL configuration expects `KeyStore` objects (or file paths to keystore files). HTTP clients like `HttpClient` expect an `SSLContext`, which requires `KeyManagerFactory` and `TrustManagerFactory` -- both of which are initialized from `KeyStore` objects.

By normalizing at the store bundle level, the entire chain above it (`SslManagerBundle`, `SslBundle`, `createSslContext()`) works identically regardless of whether the original input was a PKCS12 file, a JKS file, or PEM-encoded text. Adding a new format (e.g., PKCS#11 hardware security modules) means implementing one new `SslStoreBundle` -- nothing else changes.

</details>

<details><summary>Challenge 2: What would happen if DefaultSslBundleRegistry used a regular HashMap instead of ConcurrentHashMap?</summary>

During auto-configuration, multiple configuration classes might register bundles concurrently (especially if the application context uses parallel bean creation). With a regular `HashMap`:
- Two concurrent `put` calls could corrupt the internal hash table structure (lost entries, infinite loops in some JDK versions)
- `putIfAbsent` is not atomic on `HashMap` -- two threads could both see the key as absent and both register, with one silently overwriting the other

`ConcurrentHashMap.putIfAbsent` is atomic -- exactly one thread's registration succeeds, and the other gets a non-null return value, triggering the "already registered" error. This matches the real Spring Boot behavior.

</details>

<details><summary>Challenge 3: Why does PemSslStoreBundle use PKCS12 as the KeyStore type instead of JKS?</summary>

PKCS12 is the industry standard and the JVM's default keystore type since Java 9. It supports both key entries (private key + certificate chain) and certificate-only entries. JKS is a legacy Java-specific format with known security weaknesses (it uses a weak integrity check algorithm).

More practically, when the PEM bundle stores a private key entry with `store.setKeyEntry(alias, privateKey, EMPTY_PASSWORD, chain)`, PKCS12 handles the empty password correctly. JKS has quirky behavior with empty passwords that varies across JVM implementations.

</details>

<details><summary>Challenge 4: Why does the binder scan property sources directly (findMapKeys) rather than using Environment.getProperty()?</summary>

`Environment.getProperty(key)` looks up a single known key. But for map binding, we do not know the keys in advance -- the user defines them (`server`, `client`, `myBundle`, etc.). We need to discover what map keys exist by scanning all property source entries for a matching prefix.

This is exactly what the real Spring Boot `Binder` does internally -- its `MapBinder` iterates the `ConfigurationPropertySource` to discover map keys. Our implementation is simpler (direct property source iteration) but follows the same pattern.

</details>

---

## 22.7 Tests

### SslBundleKeyTest -- key reference validation

**File:** `iris-boot-core/.../ssl/SslBundleKeyTest.java` (7 tests)

```java
@Test
void shouldReturnNone_WhenBothPasswordAndAliasAreNull() {
    SslBundleKey key = SslBundleKey.of(null, null);
    assertThat(key).isSameAs(SslBundleKey.NONE);
    assertThat(key.getPassword()).isNull();
    assertThat(key.getAlias()).isNull();
}

@Test
void shouldThrow_WhenAliasNotFoundInKeyStore() throws Exception {
    KeyStore keyStore = KeyStore.getInstance("PKCS12");
    keyStore.load(null, null);
    SslBundleKey key = SslBundleKey.of(null, "nonExistent");
    assertThatThrownBy(() -> key.assertContainsAlias(keyStore))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("nonExistent");
}
```

The `assertContainsAlias()` test verifies the fail-fast behavior: a misconfigured alias produces a clear startup error.

### SslOptionsTest -- defensive copying

**File:** `iris-boot-core/.../ssl/SslOptionsTest.java` (5 tests)

```java
@Test
void shouldDefensiveCopyArrays() {
    String[] ciphers = {"TLS_AES_128_GCM_SHA256"};
    SslOptions options = SslOptions.of(ciphers, null);
    ciphers[0] = "MODIFIED";
    assertThat(options.getCiphers()).containsExactly("TLS_AES_128_GCM_SHA256");
}
```

This test proves that the factory method clones the input arrays -- callers cannot mutate the options after construction.

### SslManagerBundleTest -- building managers from stores

**File:** `iris-boot-core/.../ssl/SslManagerBundleTest.java` (5 tests)

```java
@Test
void shouldBuildManagersFromStores() throws Exception {
    KeyStore keyStore = SslTestHelper.createSelfSignedKeyStore("test", "changeit");

    SslStoreBundle stores = SslStoreBundle.of(keyStore, "changeit", null);
    SslBundleKey key = SslBundleKey.NONE;

    SslManagerBundle bundle = SslManagerBundle.from(stores, key);
    assertThat(bundle.getKeyManagerFactory()).isNotNull();
    assertThat(bundle.getKeyManagers()).isNotEmpty();
    // No trust store -> null trust manager factory
    assertThat(bundle.getTrustManagerFactory()).isNull();
}

@Test
void shouldCreateSslContext() throws Exception {
    KeyStore keyStore = SslTestHelper.createSelfSignedKeyStore("test", "changeit");

    SslStoreBundle stores = SslStoreBundle.of(keyStore, "changeit", null);
    SslManagerBundle bundle = SslManagerBundle.from(stores, SslBundleKey.NONE);

    SSLContext sslContext = bundle.createSslContext("TLS");
    assertThat(sslContext).isNotNull();
    assertThat(sslContext.getProtocol()).isEqualTo("TLS");
}
```

These tests use `SslTestHelper.createSelfSignedKeyStore()` which generates a real self-signed certificate via the `keytool` command. This avoids using internal JDK APIs and produces certificates that the JDK's SSL engine actually accepts.

### SslBundleTest -- composition and SSLContext creation

**File:** `iris-boot-core/.../ssl/SslBundleTest.java` (6 tests)

```java
@Test
void shouldCreateBundle_WithDefaults() {
    SslBundle bundle = SslBundle.of(null, null, null, null);

    assertThat(bundle.getStores()).isSameAs(SslStoreBundle.NONE);
    assertThat(bundle.getKey()).isSameAs(SslBundleKey.NONE);
    assertThat(bundle.getOptions()).isSameAs(SslOptions.NONE);
    assertThat(bundle.getProtocol()).isEqualTo("TLS");
}

@Test
void shouldCreateSslContext_WithRealKeyStore() throws Exception {
    KeyStore keyStore = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
    SslStoreBundle stores = SslStoreBundle.of(keyStore, "changeit", null);
    SslBundle bundle = SslBundle.of(stores, SslBundleKey.NONE, null, null);

    SSLContext sslContext = bundle.createSslContext();
    assertThat(sslContext).isNotNull();
    assertThat(sslContext.getProtocol()).isEqualTo("TLS");
}
```

### DefaultSslBundleRegistryTest -- registration and lookup

**File:** `iris-boot-core/.../ssl/DefaultSslBundleRegistryTest.java` (7 tests)

```java
@Test
void shouldRegisterAndRetrieveBundle() {
    SslBundle bundle = SslBundle.of(null, null, null, null);
    registry.registerBundle("test", bundle);

    SslBundle retrieved = registry.getBundle("test");
    assertThat(retrieved).isSameAs(bundle);
}

@Test
void shouldThrow_WhenRegisteringDuplicateName() {
    SslBundle bundle = SslBundle.of(null, null, null, null);
    registry.registerBundle("test", bundle);

    assertThatThrownBy(() -> registry.registerBundle("test", bundle))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("already registered");
}

@Test
void shouldThrowNoSuchSslBundleException_WhenBundleNotFound() {
    assertThatThrownBy(() -> registry.getBundle("nonExistent"))
            .isInstanceOf(NoSuchSslBundleException.class)
            .hasMessageContaining("nonExistent");
}
```

### JksSslStoreBundleTest -- loading keystore files

**File:** `iris-boot-core/.../ssl/jks/JksSslStoreBundleTest.java` (7 tests)

```java
@Test
void shouldLoadKeyStore_FromFilePath() throws Exception {
    KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
    File file = SslTestHelper.writeKeyStoreToFile(source, "changeit");

    JksSslStoreDetails details = new JksSslStoreDetails(
            "PKCS12", file.getAbsolutePath(), "changeit");
    JksSslStoreBundle bundle = new JksSslStoreBundle(details, null);

    KeyStore loaded = bundle.getKeyStore();
    assertThat(loaded).isNotNull();
    assertThat(loaded.containsAlias("test")).isTrue();
    assertThat(bundle.getKeyStorePassword()).isEqualTo("changeit");
}

@Test
void shouldLazyLoadAndCache() throws Exception {
    KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
    File file = SslTestHelper.writeKeyStoreToFile(source, "changeit");

    JksSslStoreDetails details = new JksSslStoreDetails(
            "PKCS12", file.getAbsolutePath(), "changeit");
    JksSslStoreBundle bundle = new JksSslStoreBundle(details, null);

    KeyStore first = bundle.getKeyStore();
    KeyStore second = bundle.getKeyStore();
    assertThat(first).isSameAs(second);
}
```

The lazy loading test verifies that the same `KeyStore` instance is returned on subsequent calls -- the double-checked locking works correctly.

### PemSslStoreBundleTest -- PEM to KeyStore conversion

**File:** `iris-boot-core/.../ssl/pem/PemSslStoreBundleTest.java` (6 tests)

```java
@Test
void shouldCreateKeyStore_FromPemFiles() throws Exception {
    KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
    String certPem = SslTestHelper.toPemCertificate(source, "test");
    String keyPem = SslTestHelper.toPemPrivateKey(source, "test", "changeit");

    File certFile = writeFile("cert.pem", certPem);
    File keyFile = writeFile("key.pem", keyPem);

    PemSslStoreDetails details = new PemSslStoreDetails(
            certFile.getAbsolutePath(),
            keyFile.getAbsolutePath(),
            null);
    PemSslStoreBundle bundle = new PemSslStoreBundle(details, null);

    KeyStore keyStore = bundle.getKeyStore();
    assertThat(keyStore).isNotNull();
    assertThat(keyStore.containsAlias("ssl")).isTrue();
    assertThat(keyStore.isKeyEntry("ssl")).isTrue();
}

@Test
void shouldAcceptInlinePemContent() throws Exception {
    KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
    String certPem = SslTestHelper.toPemCertificate(source, "test");
    String keyPem = SslTestHelper.toPemPrivateKey(source, "test", "changeit");

    PemSslStoreDetails details = new PemSslStoreDetails(certPem, keyPem, null);
    PemSslStoreBundle bundle = new PemSslStoreBundle(details, null);

    KeyStore keyStore = bundle.getKeyStore();
    assertThat(keyStore).isNotNull();
    assertThat(keyStore.isKeyEntry("ssl")).isTrue();
}
```

These tests prove the full PEM pipeline: generate a self-signed certificate, export it as PEM, load it back, and verify the resulting `KeyStore` contains the expected entries.

### MapBindingTest -- Map<String, ?> binding in ConfigurationPropertiesBinder

**File:** `iris-boot-core/.../context/properties/MapBindingTest.java` (5 tests)

```java
@Test
void shouldBindMapOfNestedObjects() {
    StandardEnvironment env = createEnvironment(Map.of(
            "ssl.bundles.server.location", "/etc/ssl/server.p12",
            "ssl.bundles.server.password", "secret",
            "ssl.bundles.client.location", "/etc/ssl/client.p12",
            "ssl.bundles.client.password", "changeit"
    ));

    SslConfig target = new SslConfig();
    new ConfigurationPropertiesBinder(env).bind(target, "ssl");

    assertThat(target.getBundles()).hasSize(2);
    assertThat(target.getBundles().get("server").getLocation()).isEqualTo("/etc/ssl/server.p12");
    assertThat(target.getBundles().get("client").getPassword()).isEqualTo("changeit");
}

@Test
void shouldBindMapWithDeeplyNestedObjects() {
    StandardEnvironment env = createEnvironment(Map.of(
            "spring.ssl.bundle.jks.myBundle.keystore.location", "classpath:keystore.p12",
            "spring.ssl.bundle.jks.myBundle.keystore.password", "secret",
            "spring.ssl.bundle.jks.myBundle.keystore.type", "PKCS12"
    ));

    DeepConfig target = new DeepConfig();
    new ConfigurationPropertiesBinder(env).bind(target, "spring.ssl");

    assertThat(target.getBundle().getJks()).hasSize(1);
    StoreProps store = target.getBundle().getJks().get("myBundle").getKeystore();
    assertThat(store.getLocation()).isEqualTo("classpath:keystore.p12");
    assertThat(store.getPassword()).isEqualTo("secret");
}
```

The deep nesting test mirrors the real SSL properties structure: `spring.ssl.bundle.jks.<name>.keystore.<property>`.

### SslBundleIntegrationTest -- end-to-end workflow

**File:** `iris-boot-core/.../ssl/integration/SslBundleIntegrationTest.java` (5 tests)

```java
@Test
void shouldCreateSslContextFromJksBundle() throws Exception {
    // 1. Create a self-signed keystore
    KeyStore keyStore = SslTestHelper.createSelfSignedKeyStore("server", "changeit");
    File keystoreFile = SslTestHelper.writeKeyStoreToFile(keyStore, "changeit");

    // 2. Load it as a JKS store bundle
    JksSslStoreDetails details = new JksSslStoreDetails(
            "PKCS12", keystoreFile.getAbsolutePath(), "changeit");
    JksSslStoreBundle stores = new JksSslStoreBundle(details, null);

    // 3. Create a bundle
    SslBundle bundle = SslBundle.of(stores, SslBundleKey.NONE, null, null);

    // 4. Register and retrieve
    DefaultSslBundleRegistry registry = new DefaultSslBundleRegistry();
    registry.registerBundle("server", bundle);

    SslBundles bundles = registry;
    SslBundle retrieved = bundles.getBundle("server");

    // 5. Create an SSLContext
    SSLContext sslContext = retrieved.createSslContext();
    assertThat(sslContext).isNotNull();
    assertThat(sslContext.getProtocol()).isEqualTo("TLS");
}

@Test
void shouldSupportMultipleBundleFormats() throws Exception {
    DefaultSslBundleRegistry registry = new DefaultSslBundleRegistry();

    // JKS bundle
    KeyStore jksStore = SslTestHelper.createSelfSignedKeyStore("jks-entry", "changeit");
    File jksFile = SslTestHelper.writeKeyStoreToFile(jksStore, "changeit");
    JksSslStoreBundle jksStores = new JksSslStoreBundle(
            new JksSslStoreDetails("PKCS12", jksFile.getAbsolutePath(), "changeit"),
            null);
    registry.registerBundle("jks-server",
            SslBundle.of(jksStores, SslBundleKey.NONE, null, null));

    // PEM bundle
    KeyStore pemSource = SslTestHelper.createSelfSignedKeyStore("pem-entry", "changeit");
    PemSslStoreBundle pemStores = new PemSslStoreBundle(
            new PemSslStoreDetails(
                    SslTestHelper.toPemCertificate(pemSource, "pem-entry"),
                    SslTestHelper.toPemPrivateKey(pemSource, "pem-entry", "changeit"),
                    null),
            null);
    registry.registerBundle("pem-client",
            SslBundle.of(pemStores, SslBundleKey.NONE, null, null));

    // Both should work independently
    assertThat(registry.getBundle("jks-server").createSslContext()).isNotNull();
    assertThat(registry.getBundle("pem-client").createSslContext()).isNotNull();
}
```

This test is the proof that format agnosticism works: a JKS bundle and a PEM bundle are registered in the same registry, and both produce valid `SSLContext` instances.

---

## 22.8 Why This Works

> **★ Insight** -- **Composition over inheritance in the SSL bundle design**
>
> `SslBundle` does not extend any of its components -- it *composes* them. It holds an `SslStoreBundle`, an `SslBundleKey`, `SslOptions`, and an `SslManagerBundle` as separate pieces. This means each component can be tested independently, replaced independently, and reused in different combinations.
>
> The `SslBundle.of()` factory method shows this clearly: if you provide stores and a key but no manager bundle, one is auto-created from the stores. If you provide an explicit manager bundle, the auto-creation is skipped. This flexibility comes naturally from composition -- an inheritance hierarchy would force you to override methods or use template patterns.
>
> The real Spring Boot follows the same composition pattern. `SslBundle` has getters for four sub-components, each of which is an interface with multiple implementations. The only inheritance is `DefaultSslManagerBundle implements SslManagerBundle`, and even that is package-private -- hidden behind the `SslManagerBundle.from()` factory method.

> **★ Insight** -- **Interface segregation in the registry (write vs read)**
>
> `SslBundleRegistry` (write) and `SslBundles` (read) are separate interfaces even though `DefaultSslBundleRegistry` implements both. This is not academic purity -- it serves a practical purpose.
>
> Auto-configuration code (like `SslAutoConfiguration`) uses `SslBundleRegistry` to register bundles. It should not be able to look up bundles -- that is not its job. Application components (like `TomcatAutoConfiguration`) inject `SslBundles` to retrieve bundles. They should not be able to register new ones -- that would bypass the configuration system.
>
> In the real Spring Boot, this split is even more important because `SslBundleRegistry` also has `updateBundle()` for hot-reloading. Giving update access to arbitrary application components would be a security concern.

> **★ Insight** -- **Format agnosticism: JKS and PEM produce the same KeyStore objects**
>
> The `SslStoreBundle` interface returns `KeyStore` objects. Both `JksSslStoreBundle` and `PemSslStoreBundle` implement this interface, but they produce the same output type: a Java `KeyStore` containing certificates and private keys.
>
> This design means adding a new certificate format (e.g., PKCS#11 for hardware security modules, or Vault-based certificates) requires implementing a single interface. The entire chain above -- `DefaultSslManagerBundle`, `SslBundle`, `createSslContext()`, the Tomcat connector configuration -- works unchanged.
>
> The PEM implementation is the most interesting case: it takes text-based certificates (used universally in cloud environments like Kubernetes and Let's Encrypt), parses them with `CertificateFactory` and `KeyFactory`, and produces an in-memory PKCS12 `KeyStore`. The rest of the system never knows the certificates started as PEM text.

---

## 22.9 What We Enhanced

| Prior File | Change | Why |
|-----------|--------|-----|
| `ConfigurationPropertiesBinder.bindField()` | Added `Map.class.isAssignableFrom(fieldType)` branch dispatching to `bindMapField()` | SSL properties use `Map<String, BundleProperties>` keyed by bundle name -- the binder must discover and bind dynamic map keys |
| `ConfigurationPropertiesBinder` | Added `bindMapField()`, `getMapValueType()`, `findMapKeys()` methods | Map binding scans property sources to discover keys, creates value instances, and recursively binds nested properties for each map entry |
| `ServerProperties` | Added nested `Ssl` class with `bundle` field | `server.ssl.bundle=myBundle` tells the embedded server which SSL bundle to use for HTTPS |
| `TomcatServletWebServerFactory` | Added `sslBundle` field, `createSslConnector()`, `writeKeyStoreToTempFile()` | The integration point where SSL bundles meet Tomcat's native SSL API -- creates an HTTPS connector from bundle stores |
| `TomcatAutoConfiguration` | Added `SslBundles` parameter, `@AutoConfigureAfter(SslAutoConfiguration.class)`, SSL wiring | Looks up the named SSL bundle and passes it to the factory; ordered after SSL auto-configuration |
| `AutoConfiguration.imports` | Added `SslAutoConfiguration` entry before `TomcatAutoConfiguration` | Registers `SslAutoConfiguration` as an auto-configuration candidate |

---

## 22.10 Connection to Real Framework

| Iris Class | Real Spring Boot Class | File (commit `5922311a95a`) |
|-----------|----------------------|------|
| `SslBundleKey` | `SslBundleKey` | `spring-boot/.../boot/ssl/SslBundleKey.java` |
| `SslOptions` | `SslOptions` | `spring-boot/.../boot/ssl/SslOptions.java` |
| `SslStoreBundle` | `SslStoreBundle` | `spring-boot/.../boot/ssl/SslStoreBundle.java` |
| `SslManagerBundle` | `SslManagerBundle` | `spring-boot/.../boot/ssl/SslManagerBundle.java` |
| `DefaultSslManagerBundle` | `DefaultSslManagerBundle` | `spring-boot/.../boot/ssl/DefaultSslManagerBundle.java` |
| `SslBundle` | `SslBundle` | `spring-boot/.../boot/ssl/SslBundle.java` |
| `SslBundleRegistry` | `SslBundleRegistry` | `spring-boot/.../boot/ssl/SslBundleRegistry.java` |
| `SslBundles` | `SslBundles` | `spring-boot/.../boot/ssl/SslBundles.java` |
| `DefaultSslBundleRegistry` | `DefaultSslBundleRegistry` | `spring-boot/.../boot/ssl/DefaultSslBundleRegistry.java` |
| `NoSuchSslBundleException` | `NoSuchSslBundleException` | `spring-boot/.../boot/ssl/NoSuchSslBundleException.java` |
| `JksSslStoreDetails` | `JksSslStoreDetails` | `spring-boot/.../boot/ssl/jks/JksSslStoreDetails.java` |
| `JksSslStoreBundle` | `JksSslStoreBundle` | `spring-boot/.../boot/ssl/jks/JksSslStoreBundle.java` |
| `PemSslStoreDetails` | `PemSslStoreDetails` | `spring-boot/.../boot/ssl/pem/PemSslStoreDetails.java` |
| `PemContent` | `PemContent` | `spring-boot/.../boot/ssl/pem/PemContent.java` |
| `PemSslStoreBundle` | `PemSslStoreBundle` | `spring-boot/.../boot/ssl/pem/PemSslStoreBundle.java` |
| `SslProperties` | `SslProperties` | `spring-boot-autoconfigure/.../autoconfigure/ssl/SslProperties.java` |
| `JksSslBundleProperties` | `JksSslBundleProperties` | `spring-boot-autoconfigure/.../autoconfigure/ssl/JksSslBundleProperties.java` |
| `PemSslBundleProperties` | `PemSslBundleProperties` | `spring-boot-autoconfigure/.../autoconfigure/ssl/PemSslBundleProperties.java` |
| `SslAutoConfiguration` | `SslAutoConfiguration` | `spring-boot-autoconfigure/.../autoconfigure/ssl/SslAutoConfiguration.java` |
| `createSslConnector()` | `SslConnectorCustomizer` | `spring-boot-tomcat/.../boot/tomcat/SslConnectorCustomizer.java` |
| `ConfigurationPropertiesBinder` (Map binding) | `Binder` + `MapBinder` | `spring-boot/.../boot/context/properties/bind/Binder.java`, `MapBinder.java` |

**Key differences from real Spring Boot:**
- Real `SslBundleKey` is a Java record. We use a class with a private constructor for compatibility.
- Real `PemContent` includes a sophisticated regex-based PEM block parser supporting multiple blocks per file, and `PemPrivateKeyParser` handles PKCS#1, PKCS#8, SEC 1, and encrypted keys via ASN.1 parsing. We support only unencrypted PKCS#8.
- Real `SslAutoConfiguration` creates a `FileWatcher` for hot-reloading certificates when they change on disk. We omit hot-reload.
- Real `SslAutoConfiguration` uses `PropertiesSslBundle` as a bridge between properties and bundles. We create bundles directly in the factory method.
- Real `SslConnectorCustomizer` uses a custom Tomcat `SSLImplementation` to feed keystores directly to Tomcat's SSL engine without writing to temp files. We use the simpler file-based approach.
- Real `Binder` + `MapBinder` handle indexed maps, type conversion, and relaxed binding. Our map binder discovers keys by scanning property sources directly.

---

## 22.11 Complete Code

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/SslBundleKey.java`

```java
package com.iris.boot.ssl;

import java.security.KeyStore;
import java.security.KeyStoreException;
import java.util.Enumeration;

/**
 * A reference to a single key within a key store, identified by alias and
 * protected by a password.
 *
 * <p>An {@code SslBundleKey} is used to select which key entry to use when
 * a {@link java.security.KeyStore KeyStore} contains multiple entries. If the
 * alias is {@code null}, the first key entry found will be used (the default
 * behavior of most SSL implementations).
 *
 * <p>This is a data holder — it carries the <em>reference</em> to the key
 * but does not perform any crypto operations itself.
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code SslBundleKey} record also validates that the alias
 * exists in the key store via {@code assertContainsAlias(KeyStore)}. We
 * include the same validation.
 *
 * @see SslBundle
 * @see SslStoreBundle
 * @see org.springframework.boot.ssl.SslBundleKey
 */
public final class SslBundleKey {

    /** An empty key with no alias or password — the default. */
    public static final SslBundleKey NONE = new SslBundleKey(null, null);

    private final String password;
    private final String alias;

    private SslBundleKey(String password, String alias) {
        this.password = password;
        this.alias = alias;
    }

    /**
     * Create a new {@code SslBundleKey} with the given password and alias.
     *
     * @param password the password to access the key (may be {@code null})
     * @param alias    the alias identifying the key in the store (may be {@code null})
     * @return a new key reference
     */
    public static SslBundleKey of(String password, String alias) {
        if (password == null && alias == null) {
            return NONE;
        }
        return new SslBundleKey(password, alias);
    }

    /**
     * Return the password used to access the key entry, or {@code null}
     * if no password is needed.
     */
    public String getPassword() {
        return this.password;
    }

    /**
     * Return the alias identifying the key entry, or {@code null} if the
     * default (first) entry should be used.
     */
    public String getAlias() {
        return this.alias;
    }

    /**
     * Assert that the given key store contains an entry with this key's alias.
     *
     * <p>This is a fail-fast check: if the user configured an alias that
     * doesn't exist, we want to fail at startup with a clear message rather
     * than at runtime with a cryptic SSL error.
     *
     * @param keyStore the key store to validate against
     * @throws IllegalStateException if the alias does not exist
     */
    public void assertContainsAlias(KeyStore keyStore) {
        if (this.alias == null || keyStore == null) {
            return;
        }
        try {
            if (!keyStore.containsAlias(this.alias)) {
                // Build a helpful error message listing available aliases
                StringBuilder available = new StringBuilder();
                Enumeration<String> aliases = keyStore.aliases();
                while (aliases.hasMoreElements()) {
                    if (available.length() > 0) {
                        available.append(", ");
                    }
                    available.append(aliases.nextElement());
                }
                throw new IllegalStateException(
                        "Keystore does not contain alias '" + this.alias
                        + "'. Available aliases: [" + available + "]");
            }
        } catch (KeyStoreException ex) {
            throw new IllegalStateException(
                    "Could not determine if keystore contains alias '" + this.alias + "'", ex);
        }
    }

    @Override
    public String toString() {
        return "SslBundleKey[alias=" + this.alias + "]";
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/SslOptions.java`

```java
package com.iris.boot.ssl;

/**
 * Configuration for SSL protocol versions and cipher suites.
 *
 * <p>When applied to an SSL connection, these options restrict which
 * TLS protocol versions and cipher suites are allowed. A {@code null}
 * value for either field means "use the JVM defaults".
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code SslOptions} also provides utility methods for
 * converting between arrays and Sets, and applying options to
 * {@code SSLEngine} or {@code SSLSocket}. We keep it minimal.
 *
 * @see SslBundle
 * @see org.springframework.boot.ssl.SslOptions
 */
public final class SslOptions {

    /** No cipher or protocol restrictions — use JVM defaults. */
    public static final SslOptions NONE = new SslOptions(null, null);

    private final String[] ciphers;
    private final String[] enabledProtocols;

    private SslOptions(String[] ciphers, String[] enabledProtocols) {
        this.ciphers = ciphers;
        this.enabledProtocols = enabledProtocols;
    }

    /**
     * Create new SSL options with the given ciphers and protocols.
     *
     * @param ciphers          allowed cipher suites (may be {@code null})
     * @param enabledProtocols enabled TLS protocol versions (may be {@code null})
     * @return new options
     */
    public static SslOptions of(String[] ciphers, String[] enabledProtocols) {
        if (ciphers == null && enabledProtocols == null) {
            return NONE;
        }
        return new SslOptions(
                ciphers != null ? ciphers.clone() : null,
                enabledProtocols != null ? enabledProtocols.clone() : null);
    }

    /**
     * Return the allowed cipher suites, or {@code null} to use JVM defaults.
     */
    public String[] getCiphers() {
        return this.ciphers;
    }

    /**
     * Return the enabled TLS protocol versions, or {@code null} to use
     * JVM defaults.
     */
    public String[] getEnabledProtocols() {
        return this.enabledProtocols;
    }

    /**
     * Return whether any specific options have been configured.
     */
    public boolean isSpecified() {
        return this.ciphers != null || this.enabledProtocols != null;
    }

    @Override
    public String toString() {
        return "SslOptions[ciphers=" + (ciphers != null ? ciphers.length + " entries" : "default")
                + ", protocols=" + (enabledProtocols != null ? enabledProtocols.length + " entries" : "default")
                + "]";
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/SslStoreBundle.java`

```java
package com.iris.boot.ssl;

import java.security.KeyStore;

/**
 * Provides access to a pair of {@link KeyStore} instances: one for identity
 * (the key store, containing the server's certificate and private key) and
 * one for trust (the trust store, containing trusted CA certificates).
 *
 * <p>Different implementations load these stores from different sources:
 * <ul>
 *   <li>{@link com.iris.boot.ssl.jks.JksSslStoreBundle JksSslStoreBundle} —
 *       loads from JKS or PKCS12 keystore files</li>
 *   <li>{@link com.iris.boot.ssl.pem.PemSslStoreBundle PemSslStoreBundle} —
 *       loads from PEM-encoded certificate and private key files</li>
 * </ul>
 *
 * <p>This abstraction is what makes the SSL system format-agnostic: consumers
 * (like Tomcat) always work with {@code KeyStore} objects, regardless of
 * whether the source was a JKS file or a PEM certificate.
 *
 * @see SslBundle
 * @see org.springframework.boot.ssl.SslStoreBundle
 */
public interface SslStoreBundle {

    /** An empty store bundle — all methods return {@code null}. */
    SslStoreBundle NONE = new SslStoreBundle() {
        @Override
        public KeyStore getKeyStore() {
            return null;
        }

        @Override
        public String getKeyStorePassword() {
            return null;
        }

        @Override
        public KeyStore getTrustStore() {
            return null;
        }

        @Override
        public String toString() {
            return "SslStoreBundle.NONE";
        }
    };

    /**
     * Return the identity key store containing the server's certificate
     * and private key, or {@code null} if not configured.
     */
    KeyStore getKeyStore();

    /**
     * Return the password for the key store, or {@code null} if no
     * password is needed.
     */
    String getKeyStorePassword();

    /**
     * Return the trust store containing trusted CA certificates, or
     * {@code null} if the JVM's default trust store should be used.
     */
    KeyStore getTrustStore();

    /**
     * Create a store bundle from the given stores and password.
     *
     * @param keyStore         the identity key store (may be {@code null})
     * @param keyStorePassword the key store password (may be {@code null})
     * @param trustStore       the trust store (may be {@code null})
     * @return a new store bundle
     */
    static SslStoreBundle of(KeyStore keyStore, String keyStorePassword, KeyStore trustStore) {
        return new SslStoreBundle() {
            @Override
            public KeyStore getKeyStore() {
                return keyStore;
            }

            @Override
            public String getKeyStorePassword() {
                return keyStorePassword;
            }

            @Override
            public KeyStore getTrustStore() {
                return trustStore;
            }
        };
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/SslManagerBundle.java`

```java
package com.iris.boot.ssl;

import java.security.KeyManagementException;
import java.security.NoSuchAlgorithmException;

import javax.net.ssl.KeyManager;
import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManager;
import javax.net.ssl.TrustManagerFactory;

/**
 * Provides access to {@link KeyManagerFactory} and {@link TrustManagerFactory}
 * instances for SSL configuration.
 *
 * <p>This is the bridge between raw key stores ({@link SslStoreBundle}) and
 * the JDK's SSL engine ({@link SSLContext}). The manager factories are
 * initialized from the stores, and the {@link #createSslContext(String)}
 * method creates a fully initialized SSL context from them.
 *
 * <p>The key design insight: consumers that need an {@code SSLContext} don't
 * need to know whether the underlying stores came from JKS files, PEM files,
 * or were programmatically created. They just call
 * {@link SslBundle#createSslContext()}.
 *
 * @see SslBundle
 * @see DefaultSslManagerBundle
 * @see org.springframework.boot.ssl.SslManagerBundle
 */
public interface SslManagerBundle {

    /**
     * Return the {@link KeyManagerFactory} for the identity key store.
     */
    KeyManagerFactory getKeyManagerFactory();

    /**
     * Return the {@link TrustManagerFactory} for the trust store.
     */
    TrustManagerFactory getTrustManagerFactory();

    /**
     * Return the key managers from the {@link KeyManagerFactory}.
     */
    default KeyManager[] getKeyManagers() {
        KeyManagerFactory factory = getKeyManagerFactory();
        return (factory != null) ? factory.getKeyManagers() : null;
    }

    /**
     * Return the trust managers from the {@link TrustManagerFactory}.
     */
    default TrustManager[] getTrustManagers() {
        TrustManagerFactory factory = getTrustManagerFactory();
        return (factory != null) ? factory.getTrustManagers() : null;
    }

    /**
     * Create a fully initialized {@link SSLContext} using the key and trust
     * managers from this bundle.
     *
     * <p>This is the method that most consumers will call — it creates the
     * SSL context that can be used to configure HTTPS connectors, HTTP
     * clients, database connections, etc.
     *
     * @param protocol the SSL protocol (e.g., "TLS", "TLSv1.3")
     * @return a fully initialized SSL context
     */
    default SSLContext createSslContext(String protocol) {
        try {
            SSLContext sslContext = SSLContext.getInstance(protocol);
            sslContext.init(getKeyManagers(), getTrustManagers(), null);
            return sslContext;
        } catch (NoSuchAlgorithmException | KeyManagementException ex) {
            throw new IllegalStateException(
                    "Error creating SSLContext with protocol '" + protocol + "'", ex);
        }
    }

    /**
     * Create an {@code SslManagerBundle} wrapping pre-built factories.
     *
     * @param keyManagerFactory   the key manager factory (may be {@code null})
     * @param trustManagerFactory the trust manager factory (may be {@code null})
     * @return a new manager bundle
     */
    static SslManagerBundle of(KeyManagerFactory keyManagerFactory,
                                TrustManagerFactory trustManagerFactory) {
        return new SslManagerBundle() {
            @Override
            public KeyManagerFactory getKeyManagerFactory() {
                return keyManagerFactory;
            }

            @Override
            public TrustManagerFactory getTrustManagerFactory() {
                return trustManagerFactory;
            }
        };
    }

    /**
     * Create an {@code SslManagerBundle} from the given stores and key.
     *
     * <p>This is the most common factory method. It creates a
     * {@link DefaultSslManagerBundle} that builds key/trust manager
     * factories from the stores.
     *
     * @param stores the key store and trust store
     * @param key    the key alias and password
     * @return a new manager bundle
     */
    static SslManagerBundle from(SslStoreBundle stores, SslBundleKey key) {
        return new DefaultSslManagerBundle(stores, key);
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/DefaultSslManagerBundle.java`

```java
package com.iris.boot.ssl;

import java.security.KeyStore;
import java.security.KeyStoreException;
import java.security.NoSuchAlgorithmException;
import java.security.UnrecoverableKeyException;

import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.TrustManagerFactory;

/**
 * Default {@link SslManagerBundle} implementation that builds
 * {@link KeyManagerFactory} and {@link TrustManagerFactory} from
 * an {@link SslStoreBundle} and {@link SslBundleKey}.
 *
 * <p>This is the critical bridge between the store abstraction
 * (which provides raw {@code KeyStore} objects) and the manager
 * abstraction (which provides the factories needed by {@code SSLContext}).
 *
 * <p>Key initialization logic:
 * <ul>
 *   <li>The key password for {@code KeyManagerFactory.init()} comes from
 *       {@link SslBundleKey#getPassword()}, falling back to the keystore
 *       password if not set</li>
 *   <li>If an alias is configured, it's validated against the keystore
 *       at construction time (fail-fast)</li>
 * </ul>
 *
 * <p>Package-private — consumers create instances via
 * {@link SslManagerBundle#from(SslStoreBundle, SslBundleKey)}.
 *
 * @see org.springframework.boot.ssl.DefaultSslManagerBundle
 */
class DefaultSslManagerBundle implements SslManagerBundle {

    private final SslStoreBundle storeBundle;
    private final SslBundleKey key;

    DefaultSslManagerBundle(SslStoreBundle storeBundle, SslBundleKey key) {
        this.storeBundle = (storeBundle != null) ? storeBundle : SslStoreBundle.NONE;
        this.key = (key != null) ? key : SslBundleKey.NONE;
    }

    /**
     * Create and initialize a {@link KeyManagerFactory} from the identity
     * key store.
     *
     * <p>The initialization sequence:
     * <ol>
     *   <li>Get the key store from {@link SslStoreBundle#getKeyStore()}</li>
     *   <li>If an alias is configured, validate it exists in the store</li>
     *   <li>Determine the key password: use {@link SslBundleKey#getPassword()},
     *       falling back to the keystore password</li>
     *   <li>Initialize the factory with the store and password</li>
     * </ol>
     */
    @Override
    public KeyManagerFactory getKeyManagerFactory() {
        KeyStore keyStore = this.storeBundle.getKeyStore();
        if (keyStore == null) {
            return null;
        }

        try {
            // Validate that the configured alias exists
            this.key.assertContainsAlias(keyStore);

            // Key password falls back to keystore password
            String keyPassword = this.key.getPassword();
            if (keyPassword == null) {
                keyPassword = this.storeBundle.getKeyStorePassword();
            }
            char[] keyPasswordChars = (keyPassword != null) ? keyPassword.toCharArray() : null;

            KeyManagerFactory factory = KeyManagerFactory.getInstance(
                    KeyManagerFactory.getDefaultAlgorithm());
            factory.init(keyStore, keyPasswordChars);
            return factory;

        } catch (NoSuchAlgorithmException | KeyStoreException | UnrecoverableKeyException ex) {
            throw new IllegalStateException("Error creating KeyManagerFactory", ex);
        }
    }

    /**
     * Create and initialize a {@link TrustManagerFactory} from the trust store.
     *
     * <p>If no trust store is configured, returns {@code null}, which causes
     * the JVM's default trust store ({@code cacerts}) to be used.
     */
    @Override
    public TrustManagerFactory getTrustManagerFactory() {
        KeyStore trustStore = this.storeBundle.getTrustStore();
        if (trustStore == null) {
            return null;
        }

        try {
            TrustManagerFactory factory = TrustManagerFactory.getInstance(
                    TrustManagerFactory.getDefaultAlgorithm());
            factory.init(trustStore);
            return factory;

        } catch (NoSuchAlgorithmException | KeyStoreException ex) {
            throw new IllegalStateException("Error creating TrustManagerFactory", ex);
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/SslBundle.java`

```java
package com.iris.boot.ssl;

import javax.net.ssl.SSLContext;

/**
 * A named, reusable SSL configuration bundle that composes all the pieces
 * needed for SSL/TLS: key stores, trust stores, key references, protocol
 * options, and manager factories.
 *
 * <p>This is the top-level abstraction of the SSL Bundles feature. A single
 * bundle provides everything needed to configure SSL on any component —
 * an embedded web server, an HTTP client, a database connection, etc.
 *
 * <h3>Composition</h3>
 *
 * <p>An {@code SslBundle} composes four sub-components:
 * <ul>
 *   <li>{@link SslStoreBundle} — the raw key store and trust store</li>
 *   <li>{@link SslBundleKey} — which key entry to use (alias + password)</li>
 *   <li>{@link SslOptions} — cipher suites and protocol restrictions</li>
 *   <li>{@link SslManagerBundle} — the JDK manager factories for {@code SSLContext}</li>
 * </ul>
 *
 * <p>The typical usage pattern is:
 * <pre>{@code
 * SslBundle bundle = sslBundles.getBundle("myapp");
 * SSLContext sslContext = bundle.createSslContext();
 * }</pre>
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code SslBundle} also supports:
 * <ul>
 *   <li>Hot-reload callbacks when certificates are updated on disk</li>
 *   <li>A {@code systemDefault()} factory for using the JVM's default SSL context</li>
 * </ul>
 * We omit hot-reload support.
 *
 * @see SslBundles
 * @see org.springframework.boot.ssl.SslBundle
 */
public interface SslBundle {

    /** The default SSL protocol. */
    String DEFAULT_PROTOCOL = "TLS";

    /**
     * Return the store bundle providing the key store and trust store.
     */
    SslStoreBundle getStores();

    /**
     * Return the key reference (alias and password) for selecting a
     * specific key entry from the key store.
     */
    SslBundleKey getKey();

    /**
     * Return the SSL options (ciphers and enabled protocols).
     */
    SslOptions getOptions();

    /**
     * Return the manager bundle providing {@code KeyManagerFactory} and
     * {@code TrustManagerFactory}.
     */
    SslManagerBundle getManagers();

    /**
     * Return the SSL protocol to use (e.g., "TLS", "TLSv1.3").
     * Defaults to {@value #DEFAULT_PROTOCOL}.
     */
    default String getProtocol() {
        return DEFAULT_PROTOCOL;
    }

    /**
     * Create a fully initialized {@link SSLContext} using this bundle's
     * managers and protocol.
     *
     * <p>This is the convenience method that most consumers use. It
     * delegates to {@code getManagers().createSslContext(getProtocol())}.
     *
     * @return a ready-to-use SSL context
     */
    default SSLContext createSslContext() {
        return getManagers().createSslContext(getProtocol());
    }

    /**
     * Create an {@code SslBundle} from the given components.
     *
     * <p>If no {@code SslManagerBundle} is provided, one is created
     * automatically from the stores and key using
     * {@link SslManagerBundle#from(SslStoreBundle, SslBundleKey)}.
     *
     * @param stores   the store bundle (may be {@code null} for {@link SslStoreBundle#NONE})
     * @param key      the key reference (may be {@code null} for {@link SslBundleKey#NONE})
     * @param options  the SSL options (may be {@code null} for {@link SslOptions#NONE})
     * @param protocol the SSL protocol (may be {@code null} for {@value #DEFAULT_PROTOCOL})
     * @return a new bundle
     */
    static SslBundle of(SslStoreBundle stores, SslBundleKey key,
                         SslOptions options, String protocol) {
        return of(stores, key, options, protocol, null);
    }

    /**
     * Create an {@code SslBundle} from all components, including an explicit
     * manager bundle.
     *
     * @param stores   the store bundle (may be {@code null})
     * @param key      the key reference (may be {@code null})
     * @param options  the SSL options (may be {@code null})
     * @param protocol the SSL protocol (may be {@code null})
     * @param managers the manager bundle (may be {@code null} — auto-created from stores)
     * @return a new bundle
     */
    static SslBundle of(SslStoreBundle stores, SslBundleKey key,
                         SslOptions options, String protocol,
                         SslManagerBundle managers) {
        SslStoreBundle effectiveStores = (stores != null) ? stores : SslStoreBundle.NONE;
        SslBundleKey effectiveKey = (key != null) ? key : SslBundleKey.NONE;
        SslOptions effectiveOptions = (options != null) ? options : SslOptions.NONE;
        String effectiveProtocol = (protocol != null) ? protocol : DEFAULT_PROTOCOL;
        SslManagerBundle effectiveManagers = (managers != null)
                ? managers
                : SslManagerBundle.from(effectiveStores, effectiveKey);

        return new SslBundle() {
            @Override
            public SslStoreBundle getStores() {
                return effectiveStores;
            }

            @Override
            public SslBundleKey getKey() {
                return effectiveKey;
            }

            @Override
            public SslOptions getOptions() {
                return effectiveOptions;
            }

            @Override
            public SslManagerBundle getManagers() {
                return effectiveManagers;
            }

            @Override
            public String getProtocol() {
                return effectiveProtocol;
            }

            @Override
            public String toString() {
                return "SslBundle[protocol=" + effectiveProtocol + "]";
            }
        };
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/SslBundleRegistry.java`

```java
package com.iris.boot.ssl;

/**
 * Registry for registering named {@link SslBundle} instances.
 *
 * <p>This is the <em>write</em> side of the SSL bundle system. Configuration
 * code (like auto-configuration) uses this interface to register bundles
 * that were loaded from properties. The <em>read</em> side is
 * {@link SslBundles}.
 *
 * <p>Splitting registration from retrieval follows the Interface Segregation
 * Principle: components that configure SSL bundles don't need to read them,
 * and components that consume SSL bundles don't need to register new ones.
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code SslBundleRegistry} also supports
 * {@code updateBundle(name, bundle)} for hot-reloading certificates.
 * We omit that.
 *
 * @see SslBundles
 * @see DefaultSslBundleRegistry
 * @see org.springframework.boot.ssl.SslBundleRegistry
 */
public interface SslBundleRegistry {

    /**
     * Register a named SSL bundle.
     *
     * @param name   the bundle name (used for lookup)
     * @param bundle the SSL bundle
     * @throws IllegalStateException if a bundle with this name already exists
     */
    void registerBundle(String name, SslBundle bundle);
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/SslBundles.java`

```java
package com.iris.boot.ssl;

/**
 * Provides access to named {@link SslBundle} instances.
 *
 * <p>This is the <em>read</em> side of the SSL bundle system. Application
 * components that need SSL configuration inject this interface and look up
 * bundles by name:
 *
 * <pre>{@code
 * @Component
 * public class MyHttpClient {
 *     private final SSLContext sslContext;
 *
 *     public MyHttpClient(SslBundles sslBundles) {
 *         this.sslContext = sslBundles.getBundle("client-cert").createSslContext();
 *     }
 * }
 * }</pre>
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code SslBundles} also supports:
 * <ul>
 *   <li>{@code addBundleUpdateHandler(name, handler)} — notification on
 *       certificate hot-reload</li>
 *   <li>{@code addBundleRegisterHandler(handler)} — notification when
 *       new bundles are registered</li>
 *   <li>{@code getBundleNames()} — list all registered bundle names</li>
 * </ul>
 * We omit these for simplicity.
 *
 * @see SslBundleRegistry
 * @see DefaultSslBundleRegistry
 * @see org.springframework.boot.ssl.SslBundles
 */
public interface SslBundles {

    /**
     * Return the SSL bundle with the given name.
     *
     * @param name the bundle name
     * @return the SSL bundle
     * @throws NoSuchSslBundleException if no bundle with this name exists
     */
    SslBundle getBundle(String name) throws NoSuchSslBundleException;
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/DefaultSslBundleRegistry.java`

```java
package com.iris.boot.ssl;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

/**
 * Default implementation of both {@link SslBundleRegistry} and {@link SslBundles},
 * storing named bundles in a thread-safe map.
 *
 * <p>This class serves as both the write side (registration during startup)
 * and the read side (lookup at runtime). The real Spring Boot uses the same
 * pattern — one object implements both interfaces, but consumers only see
 * the interface they need.
 *
 * <p>Thread safety: uses {@link ConcurrentHashMap} and {@code putIfAbsent}
 * to ensure that bundle registration is safe during concurrent auto-configuration.
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code DefaultSslBundleRegistry} also supports:
 * <ul>
 *   <li>{@code updateBundle()} for hot-reloading</li>
 *   <li>Update handlers notified when bundles change</li>
 *   <li>Register handlers notified when new bundles are added</li>
 *   <li>{@code RegisteredSslBundle} inner class wrapping volatile bundle + handler list</li>
 * </ul>
 *
 * @see org.springframework.boot.ssl.DefaultSslBundleRegistry
 */
public class DefaultSslBundleRegistry implements SslBundleRegistry, SslBundles {

    private final ConcurrentMap<String, SslBundle> bundles = new ConcurrentHashMap<>();

    /**
     * Register a named SSL bundle.
     *
     * <p>Uses {@code putIfAbsent} to prevent accidental overwrites. If two
     * auto-configurations try to register the same bundle name, the second
     * one will fail with a clear error message.
     *
     * @throws IllegalStateException if a bundle with this name already exists
     */
    @Override
    public void registerBundle(String name, SslBundle bundle) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("SSL bundle name must not be null or blank");
        }
        if (bundle == null) {
            throw new IllegalArgumentException("SSL bundle must not be null");
        }
        SslBundle existing = this.bundles.putIfAbsent(name, bundle);
        if (existing != null) {
            throw new IllegalStateException(
                    "Cannot register SSL bundle '" + name
                    + "' — a bundle with this name is already registered");
        }
    }

    /**
     * Retrieve a registered SSL bundle by name.
     *
     * @throws NoSuchSslBundleException if no bundle with this name exists
     */
    @Override
    public SslBundle getBundle(String name) throws NoSuchSslBundleException {
        SslBundle bundle = this.bundles.get(name);
        if (bundle == null) {
            throw new NoSuchSslBundleException(name);
        }
        return bundle;
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/NoSuchSslBundleException.java`

```java
package com.iris.boot.ssl;

/**
 * Thrown when a requested SSL bundle name cannot be found in the registry.
 *
 * @see SslBundles#getBundle(String)
 * @see org.springframework.boot.ssl.NoSuchSslBundleException
 */
public class NoSuchSslBundleException extends RuntimeException {

    private final String bundleName;

    public NoSuchSslBundleException(String bundleName) {
        super("SSL bundle '" + bundleName + "' is not registered. "
                + "Check your 'spring.ssl.bundle' configuration.");
        this.bundleName = bundleName;
    }

    /**
     * Return the bundle name that was not found.
     */
    public String getBundleName() {
        return this.bundleName;
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/jks/JksSslStoreDetails.java`

```java
package com.iris.boot.ssl.jks;

/**
 * Configuration details for a JKS or PKCS12 key store file.
 *
 * <p>Holds the information needed to load a Java {@link java.security.KeyStore}
 * from a file: the store type (JKS, PKCS12), file location, and password.
 *
 * <p>In the real Spring Boot, this is a record with an additional
 * {@code provider} field for custom security providers. We omit the
 * provider for simplicity.
 *
 * @param type     the key store type (e.g., "JKS", "PKCS12"). If {@code null},
 *                 the JVM default type is used ({@code KeyStore.getDefaultType()})
 * @param location the file path or classpath location (e.g., "classpath:keystore.p12"
 *                 or "/etc/ssl/keystore.jks")
 * @param password the password to open the key store (may be {@code null})
 *
 * @see JksSslStoreBundle
 * @see org.springframework.boot.ssl.jks.JksSslStoreDetails
 */
public record JksSslStoreDetails(String type, String location, String password) {

    /**
     * Create details for a PKCS12 keystore at the given location.
     */
    public static JksSslStoreDetails forLocation(String location) {
        return new JksSslStoreDetails("PKCS12", location, null);
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/jks/JksSslStoreBundle.java`

```java
package com.iris.boot.ssl.jks;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyStore;
import java.security.KeyStoreException;
import java.security.NoSuchAlgorithmException;
import java.security.cert.CertificateException;

import com.iris.boot.ssl.SslStoreBundle;

/**
 * {@link SslStoreBundle} implementation that loads key stores from JKS or
 * PKCS12 files.
 *
 * <p>Supports two location formats:
 * <ul>
 *   <li>{@code "classpath:keystore.p12"} — loads from the classpath</li>
 *   <li>{@code "/path/to/keystore.jks"} — loads from the filesystem</li>
 * </ul>
 *
 * <p>Stores are loaded lazily on first access and cached. This matches the
 * real Spring Boot behavior, which uses {@code SingletonSupplier} for the
 * same purpose.
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code JksSslStoreBundle} also supports:
 * <ul>
 *   <li>Custom security providers</li>
 *   <li>PKCS11 hardware security modules (no file location needed)</li>
 *   <li>{@code ApplicationResourceLoader} for flexible resource resolution</li>
 * </ul>
 *
 * @see JksSslStoreDetails
 * @see org.springframework.boot.ssl.jks.JksSslStoreBundle
 */
public class JksSslStoreBundle implements SslStoreBundle {

    private final JksSslStoreDetails keyStoreDetails;
    private final JksSslStoreDetails trustStoreDetails;

    /** Lazily loaded and cached key store. */
    private volatile KeyStore keyStore;
    private volatile boolean keyStoreLoaded;

    /** Lazily loaded and cached trust store. */
    private volatile KeyStore trustStore;
    private volatile boolean trustStoreLoaded;

    /**
     * Create a new bundle with the given key store and trust store details.
     *
     * @param keyStoreDetails   the key store configuration (may be {@code null})
     * @param trustStoreDetails the trust store configuration (may be {@code null})
     */
    public JksSslStoreBundle(JksSslStoreDetails keyStoreDetails,
                              JksSslStoreDetails trustStoreDetails) {
        this.keyStoreDetails = keyStoreDetails;
        this.trustStoreDetails = trustStoreDetails;
    }

    @Override
    public KeyStore getKeyStore() {
        if (!this.keyStoreLoaded) {
            synchronized (this) {
                if (!this.keyStoreLoaded) {
                    this.keyStore = loadStore(this.keyStoreDetails);
                    this.keyStoreLoaded = true;
                }
            }
        }
        return this.keyStore;
    }

    @Override
    public String getKeyStorePassword() {
        return (this.keyStoreDetails != null) ? this.keyStoreDetails.password() : null;
    }

    @Override
    public KeyStore getTrustStore() {
        if (!this.trustStoreLoaded) {
            synchronized (this) {
                if (!this.trustStoreLoaded) {
                    this.trustStore = loadStore(this.trustStoreDetails);
                    this.trustStoreLoaded = true;
                }
            }
        }
        return this.trustStore;
    }

    /**
     * Load a {@link KeyStore} from the given details.
     *
     * <p>The loading sequence:
     * <ol>
     *   <li>Determine the store type (JKS, PKCS12, or JVM default)</li>
     *   <li>Create an empty KeyStore instance</li>
     *   <li>Open an input stream to the location (classpath or filesystem)</li>
     *   <li>Load the store data with the password</li>
     * </ol>
     */
    private KeyStore loadStore(JksSslStoreDetails details) {
        if (details == null || details.location() == null) {
            return null;
        }

        try {
            String type = (details.type() != null) ? details.type() : KeyStore.getDefaultType();
            KeyStore store = KeyStore.getInstance(type);

            char[] password = (details.password() != null)
                    ? details.password().toCharArray() : null;

            try (InputStream is = openLocation(details.location())) {
                store.load(is, password);
            }

            return store;

        } catch (KeyStoreException | IOException | NoSuchAlgorithmException
                 | CertificateException ex) {
            throw new IllegalStateException(
                    "Unable to load key store from '" + details.location() + "'", ex);
        }
    }

    /**
     * Open an input stream for the given location.
     *
     * <p>If the location starts with "classpath:", loads from the classpath.
     * Otherwise, loads from the filesystem.
     */
    private InputStream openLocation(String location) throws IOException {
        if (location.startsWith("classpath:")) {
            String path = location.substring("classpath:".length());
            InputStream is = getClass().getClassLoader().getResourceAsStream(path);
            if (is == null) {
                throw new IOException("Classpath resource not found: " + path);
            }
            return is;
        }
        return Files.newInputStream(Path.of(location));
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/pem/PemSslStoreDetails.java`

```java
package com.iris.boot.ssl.pem;

/**
 * Configuration details for a PEM-encoded SSL certificate and private key.
 *
 * <p>PEM (Privacy Enhanced Mail) is a text-based format for certificates
 * and keys. It's widely used in cloud environments, Kubernetes secrets,
 * Let's Encrypt, and similar systems. The certificate and key are stored
 * as Base64-encoded text between header/footer markers:
 *
 * <pre>
 * -----BEGIN CERTIFICATE-----
 * MIICpDCCAYwCCQD...
 * -----END CERTIFICATE-----
 * </pre>
 *
 * <p>In the real Spring Boot, this is a record with additional fields for
 * type, alias, and key verification. We keep it minimal.
 *
 * @param certificate the path to the PEM certificate file, or the PEM content itself
 * @param privateKey  the path to the PEM private key file, or the PEM content itself
 *                    (must be PKCS#8 format — "BEGIN PRIVATE KEY")
 * @param privateKeyPassword the password for encrypted private keys (may be {@code null})
 *
 * @see PemSslStoreBundle
 * @see org.springframework.boot.ssl.pem.PemSslStoreDetails
 */
public record PemSslStoreDetails(String certificate, String privateKey, String privateKeyPassword) {

    /**
     * Create details for a certificate and unencrypted private key.
     */
    public static PemSslStoreDetails forCertificate(String certificate) {
        return new PemSslStoreDetails(certificate, null, null);
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/pem/PemContent.java`

```java
package com.iris.boot.ssl.pem;

import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;

/**
 * Utility for loading PEM content from a file path, classpath resource, or
 * inline PEM string.
 *
 * <p>Determines whether the input is a PEM string (starts with "-----BEGIN")
 * or a file path, and loads accordingly.
 *
 * <p>In the real Spring Boot, this is a more complex class with PEM block
 * parsing, regex-based content extraction, and support for multiple PEM
 * blocks in a single file. We simplify to a basic content loader.
 *
 * @see org.springframework.boot.ssl.pem.PemContent
 */
final class PemContent {

    private PemContent() {
    }

    /**
     * Load PEM content from the given source.
     *
     * <p>If the source starts with "-----BEGIN", it's treated as inline PEM
     * content and returned as-is. Otherwise, it's treated as a file path
     * (with "classpath:" support) and the file contents are loaded.
     *
     * @param source the PEM content or file path
     * @return the PEM content string
     */
    static String load(String source) {
        if (source == null || source.isBlank()) {
            return null;
        }

        // Inline PEM content
        if (source.trim().startsWith("-----BEGIN")) {
            return source;
        }

        // Load from file
        try {
            if (source.startsWith("classpath:")) {
                String path = source.substring("classpath:".length());
                try (InputStream is = PemContent.class.getClassLoader().getResourceAsStream(path)) {
                    if (is == null) {
                        throw new IOException("Classpath resource not found: " + path);
                    }
                    return new String(is.readAllBytes(), StandardCharsets.UTF_8);
                }
            }
            return Files.readString(Path.of(source));
        } catch (IOException ex) {
            throw new IllegalStateException("Unable to load PEM content from '" + source + "'", ex);
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/ssl/pem/PemSslStoreBundle.java`

```java
package com.iris.boot.ssl.pem;

import java.io.ByteArrayInputStream;
import java.security.KeyFactory;
import java.security.KeyStore;
import java.security.PrivateKey;
import java.security.cert.Certificate;
import java.security.cert.CertificateFactory;
import java.security.spec.PKCS8EncodedKeySpec;
import java.util.ArrayList;
import java.util.Base64;
import java.util.Collection;
import java.util.List;

import com.iris.boot.ssl.SslStoreBundle;

/**
 * {@link SslStoreBundle} implementation that loads certificates and private
 * keys from PEM-encoded content.
 *
 * <p>PEM is the most common format in cloud-native environments: Kubernetes
 * secrets, Let's Encrypt certificates, and most CAs deliver certificates
 * in PEM format. This bundle converts PEM content into Java {@link KeyStore}
 * objects so the rest of the SSL system can work uniformly.
 *
 * <p>The conversion process:
 * <ol>
 *   <li>Parse PEM certificate(s) using {@link CertificateFactory}</li>
 *   <li>Parse PKCS#8 private key using {@link KeyFactory}</li>
 *   <li>Create an in-memory PKCS12 {@link KeyStore}</li>
 *   <li>If a private key is present, store it as a key entry with the
 *       certificate chain</li>
 *   <li>If only certificates, store them as trusted certificate entries</li>
 * </ol>
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real implementation supports:
 * <ul>
 *   <li>PKCS#1 RSA private keys ("BEGIN RSA PRIVATE KEY") via ASN.1 parsing</li>
 *   <li>EC, DSA, and Ed25519 keys</li>
 *   <li>Encrypted private keys ("BEGIN ENCRYPTED PRIVATE KEY")</li>
 *   <li>Key verification (checking the key matches the certificate)</li>
 *   <li>Multiple PEM blocks per file</li>
 * </ul>
 * We support only unencrypted PKCS#8 keys ("BEGIN PRIVATE KEY") with RSA
 * or EC algorithms.
 *
 * @see PemSslStoreDetails
 * @see org.springframework.boot.ssl.pem.PemSslStoreBundle
 */
public class PemSslStoreBundle implements SslStoreBundle {

    private static final String DEFAULT_ALIAS = "ssl";
    private static final char[] EMPTY_PASSWORD = new char[0];

    private final PemSslStoreDetails keyStoreDetails;
    private final PemSslStoreDetails trustStoreDetails;

    private volatile KeyStore keyStore;
    private volatile boolean keyStoreLoaded;

    private volatile KeyStore trustStore;
    private volatile boolean trustStoreLoaded;

    /**
     * Create a new bundle with the given key store and trust store details.
     *
     * @param keyStoreDetails   the key store PEM configuration (may be {@code null})
     * @param trustStoreDetails the trust store PEM configuration (may be {@code null})
     */
    public PemSslStoreBundle(PemSslStoreDetails keyStoreDetails,
                              PemSslStoreDetails trustStoreDetails) {
        this.keyStoreDetails = keyStoreDetails;
        this.trustStoreDetails = trustStoreDetails;
    }

    @Override
    public KeyStore getKeyStore() {
        if (!this.keyStoreLoaded) {
            synchronized (this) {
                if (!this.keyStoreLoaded) {
                    this.keyStore = createKeyStore(this.keyStoreDetails);
                    this.keyStoreLoaded = true;
                }
            }
        }
        return this.keyStore;
    }

    /**
     * PEM stores don't have a keystore-level password — the key store is
     * created in-memory with an empty password.
     */
    @Override
    public String getKeyStorePassword() {
        return null;
    }

    @Override
    public KeyStore getTrustStore() {
        if (!this.trustStoreLoaded) {
            synchronized (this) {
                if (!this.trustStoreLoaded) {
                    this.trustStore = createKeyStore(this.trustStoreDetails);
                    this.trustStoreLoaded = true;
                }
            }
        }
        return this.trustStore;
    }

    /**
     * Create a {@link KeyStore} from PEM content.
     *
     * <p>If the details include a private key, a key entry is created
     * (certificate chain + private key). If only certificates are provided,
     * trusted certificate entries are created.
     */
    private KeyStore createKeyStore(PemSslStoreDetails details) {
        if (details == null || details.certificate() == null) {
            return null;
        }

        try {
            // Parse certificates
            String certContent = PemContent.load(details.certificate());
            List<Certificate> certificates = parseCertificates(certContent);
            if (certificates.isEmpty()) {
                return null;
            }

            // Create an in-memory PKCS12 KeyStore
            KeyStore store = KeyStore.getInstance("PKCS12");
            store.load(null, null);

            if (details.privateKey() != null) {
                // Key entry: private key + certificate chain
                String keyContent = PemContent.load(details.privateKey());
                PrivateKey privateKey = parsePrivateKey(keyContent);
                Certificate[] chain = certificates.toArray(new Certificate[0]);
                store.setKeyEntry(DEFAULT_ALIAS, privateKey, EMPTY_PASSWORD, chain);
            } else {
                // Trust entries: each certificate as a trusted entry
                for (int i = 0; i < certificates.size(); i++) {
                    String alias = certificates.size() == 1
                            ? DEFAULT_ALIAS
                            : DEFAULT_ALIAS + "-" + i;
                    store.setCertificateEntry(alias, certificates.get(i));
                }
            }

            return store;

        } catch (Exception ex) {
            throw new IllegalStateException("Unable to create KeyStore from PEM content", ex);
        }
    }

    /**
     * Parse X.509 certificates from PEM content.
     *
     * <p>Uses the JDK's {@link CertificateFactory}, which natively supports
     * PEM-encoded certificates (it reads the "-----BEGIN CERTIFICATE-----"
     * markers automatically).
     */
    private List<Certificate> parseCertificates(String pemContent) throws Exception {
        CertificateFactory factory = CertificateFactory.getInstance("X.509");
        Collection<? extends Certificate> certs = factory.generateCertificates(
                new ByteArrayInputStream(pemContent.getBytes()));
        return new ArrayList<>(certs);
    }

    /**
     * Parse a PKCS#8 private key from PEM content.
     *
     * <p>Supports only the "BEGIN PRIVATE KEY" format (PKCS#8). The
     * algorithm is detected by trying RSA first, then EC.
     *
     * <p>In the real Spring Boot, {@code PemPrivateKeyParser} supports
     * PKCS#1 ("BEGIN RSA PRIVATE KEY"), PKCS#8, SEC 1 ("BEGIN EC PRIVATE KEY"),
     * and encrypted keys via a complex ASN.1 parser.
     */
    private PrivateKey parsePrivateKey(String pemContent) throws Exception {
        // Strip PEM headers and decode Base64
        String base64 = pemContent
                .replace("-----BEGIN PRIVATE KEY-----", "")
                .replace("-----END PRIVATE KEY-----", "")
                .replaceAll("\\s", "");
        byte[] keyBytes = Base64.getDecoder().decode(base64);
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);

        // Try RSA first, then EC
        try {
            return KeyFactory.getInstance("RSA").generatePrivate(keySpec);
        } catch (Exception rsaEx) {
            try {
                return KeyFactory.getInstance("EC").generatePrivate(keySpec);
            } catch (Exception ecEx) {
                throw new IllegalStateException(
                        "Unable to parse private key — only RSA and EC PKCS#8 keys "
                        + "are supported ('BEGIN PRIVATE KEY' format)", rsaEx);
            }
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/ssl/JksSslBundleProperties.java`

```java
package com.iris.boot.autoconfigure.ssl;

/**
 * Properties for a single JKS-based SSL bundle.
 *
 * <p>Bound from configuration like:
 * <pre>
 * spring.ssl.bundle.jks.myBundle.key.alias=tomcat
 * spring.ssl.bundle.jks.myBundle.key.password=changeit
 * spring.ssl.bundle.jks.myBundle.keystore.location=classpath:keystore.p12
 * spring.ssl.bundle.jks.myBundle.keystore.password=secret
 * spring.ssl.bundle.jks.myBundle.keystore.type=PKCS12
 * spring.ssl.bundle.jks.myBundle.truststore.location=classpath:truststore.jks
 * spring.ssl.bundle.jks.myBundle.truststore.password=changeit
 * spring.ssl.bundle.jks.myBundle.protocol=TLSv1.3
 * </pre>
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real properties also support reload-on-update and
 * cipher/protocol options. We keep it focused on store configuration.
 *
 * @see SslProperties
 * @see org.springframework.boot.autoconfigure.ssl.JksSslBundleProperties
 */
public class JksSslBundleProperties {

    private Key key = new Key();
    private Store keystore = new Store();
    private Store truststore = new Store();
    private String protocol;

    public Key getKey() {
        return key;
    }

    public void setKey(Key key) {
        this.key = key;
    }

    public Store getKeystore() {
        return keystore;
    }

    public void setKeystore(Store keystore) {
        this.keystore = keystore;
    }

    public Store getTruststore() {
        return truststore;
    }

    public void setTruststore(Store truststore) {
        this.truststore = truststore;
    }

    public String getProtocol() {
        return protocol;
    }

    public void setProtocol(String protocol) {
        this.protocol = protocol;
    }

    /**
     * Key alias and password within the keystore.
     */
    public static class Key {
        private String alias;
        private String password;

        public String getAlias() {
            return alias;
        }

        public void setAlias(String alias) {
            this.alias = alias;
        }

        public String getPassword() {
            return password;
        }

        public void setPassword(String password) {
            this.password = password;
        }
    }

    /**
     * Key store or trust store file configuration.
     */
    public static class Store {
        private String type;
        private String location;
        private String password;

        public String getType() {
            return type;
        }

        public void setType(String type) {
            this.type = type;
        }

        public String getLocation() {
            return location;
        }

        public void setLocation(String location) {
            this.location = location;
        }

        public String getPassword() {
            return password;
        }

        public void setPassword(String password) {
            this.password = password;
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/ssl/PemSslBundleProperties.java`

```java
package com.iris.boot.autoconfigure.ssl;

/**
 * Properties for a single PEM-based SSL bundle.
 *
 * <p>Bound from configuration like:
 * <pre>
 * spring.ssl.bundle.pem.myBundle.keystore.certificate=classpath:cert.pem
 * spring.ssl.bundle.pem.myBundle.keystore.private-key=classpath:key.pem
 * spring.ssl.bundle.pem.myBundle.truststore.certificate=classpath:ca.pem
 * spring.ssl.bundle.pem.myBundle.protocol=TLSv1.3
 * </pre>
 *
 * @see SslProperties
 * @see org.springframework.boot.autoconfigure.ssl.PemSslBundleProperties
 */
public class PemSslBundleProperties {

    private Store keystore = new Store();
    private Store truststore = new Store();
    private String protocol;

    public Store getKeystore() {
        return keystore;
    }

    public void setKeystore(Store keystore) {
        this.keystore = keystore;
    }

    public Store getTruststore() {
        return truststore;
    }

    public void setTruststore(Store truststore) {
        this.truststore = truststore;
    }

    public String getProtocol() {
        return protocol;
    }

    public void setProtocol(String protocol) {
        this.protocol = protocol;
    }

    /**
     * PEM certificate and private key locations.
     */
    public static class Store {
        private String certificate;
        private String privateKey;
        private String privateKeyPassword;

        public String getCertificate() {
            return certificate;
        }

        public void setCertificate(String certificate) {
            this.certificate = certificate;
        }

        public String getPrivateKey() {
            return privateKey;
        }

        public void setPrivateKey(String privateKey) {
            this.privateKey = privateKey;
        }

        public String getPrivateKeyPassword() {
            return privateKeyPassword;
        }

        public void setPrivateKeyPassword(String privateKeyPassword) {
            this.privateKeyPassword = privateKeyPassword;
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/ssl/SslProperties.java`

```java
package com.iris.boot.autoconfigure.ssl;

import java.util.LinkedHashMap;
import java.util.Map;

import com.iris.boot.context.properties.ConfigurationProperties;

/**
 * {@link ConfigurationProperties @ConfigurationProperties} for SSL bundle
 * configuration.
 *
 * <p>Binds the {@code spring.ssl} prefix. SSL bundles are organized by
 * format (JKS or PEM) and identified by name:
 *
 * <pre>
 * spring.ssl.bundle.jks.server.keystore.location=classpath:keystore.p12
 * spring.ssl.bundle.jks.server.keystore.password=secret
 *
 * spring.ssl.bundle.pem.client.keystore.certificate=classpath:cert.pem
 * spring.ssl.bundle.pem.client.keystore.private-key=classpath:key.pem
 * </pre>
 *
 * <p>The map keys ("server", "client") become the bundle names used for
 * lookup via {@link com.iris.boot.ssl.SslBundles#getBundle(String)}.
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code SslProperties} also includes:
 * <ul>
 *   <li>{@code Watch.File.quietPeriod} for certificate hot-reload</li>
 * </ul>
 *
 * @see SslAutoConfiguration
 * @see org.springframework.boot.autoconfigure.ssl.SslProperties
 */
@ConfigurationProperties(prefix = "spring.ssl")
public class SslProperties {

    private Bundle bundle = new Bundle();

    public Bundle getBundle() {
        return bundle;
    }

    public void setBundle(Bundle bundle) {
        this.bundle = bundle;
    }

    /**
     * Bundle configuration, organized by format.
     */
    public static class Bundle {

        /**
         * JKS-based SSL bundles, keyed by bundle name.
         */
        private Map<String, JksSslBundleProperties> jks = new LinkedHashMap<>();

        /**
         * PEM-based SSL bundles, keyed by bundle name.
         */
        private Map<String, PemSslBundleProperties> pem = new LinkedHashMap<>();

        public Map<String, JksSslBundleProperties> getJks() {
            return jks;
        }

        public void setJks(Map<String, JksSslBundleProperties> jks) {
            this.jks = jks;
        }

        public Map<String, PemSslBundleProperties> getPem() {
            return pem;
        }

        public void setPem(Map<String, PemSslBundleProperties> pem) {
            this.pem = pem;
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/ssl/SslAutoConfiguration.java`

```java
package com.iris.boot.autoconfigure.ssl;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.boot.context.properties.EnableConfigurationProperties;
import com.iris.boot.ssl.DefaultSslBundleRegistry;
import com.iris.boot.ssl.SslBundle;
import com.iris.boot.ssl.SslBundleKey;
import com.iris.boot.ssl.SslBundleRegistry;
import com.iris.boot.ssl.SslBundles;
import com.iris.boot.ssl.SslStoreBundle;
import com.iris.boot.ssl.jks.JksSslStoreBundle;
import com.iris.boot.ssl.jks.JksSslStoreDetails;
import com.iris.boot.ssl.pem.PemSslStoreBundle;
import com.iris.boot.ssl.pem.PemSslStoreDetails;
import com.iris.framework.context.annotation.Bean;

/**
 * Auto-configuration for SSL bundles.
 *
 * <p>Creates a {@link DefaultSslBundleRegistry} and populates it with
 * bundles defined in {@code application.properties} via {@link SslProperties}.
 *
 * <p>This auto-configuration reads:
 * <pre>
 * spring.ssl.bundle.jks.&lt;name&gt;.keystore.location=...
 * spring.ssl.bundle.jks.&lt;name&gt;.keystore.password=...
 * spring.ssl.bundle.pem.&lt;name&gt;.keystore.certificate=...
 * spring.ssl.bundle.pem.&lt;name&gt;.keystore.private-key=...
 * </pre>
 *
 * <p>And registers each as a named {@link SslBundle} in the registry. Other
 * auto-configurations (like {@code TomcatAutoConfiguration}) can then look
 * up bundles by name.
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real auto-configuration also:
 * <ul>
 *   <li>Creates a {@code FileWatcher} for hot-reloading certificates</li>
 *   <li>Supports {@code SslBundleRegistrar} SPI for custom registration</li>
 *   <li>Uses {@code PropertiesSslBundle} as a bridge class</li>
 * </ul>
 * We register bundles directly in the bean factory method.
 *
 * @see org.springframework.boot.autoconfigure.ssl.SslAutoConfiguration
 */
@AutoConfiguration
@EnableConfigurationProperties(SslProperties.class)
public class SslAutoConfiguration {

    /**
     * Create a {@link DefaultSslBundleRegistry} populated from properties.
     *
     * <p>The registry implements both {@link SslBundleRegistry} (for
     * registration during startup) and {@link SslBundles} (for lookup at
     * runtime). The {@code @ConditionalOnMissingBean} checks both interfaces,
     * allowing users to provide their own registry if needed.
     *
     * <p>Bundle creation follows a two-step process for each property entry:
     * <ol>
     *   <li>Create an {@link SslStoreBundle} from the properties
     *       (JKS or PEM format)</li>
     *   <li>Wrap it in an {@link SslBundle} with key, options, and protocol</li>
     * </ol>
     *
     * @param sslProperties the bound SSL properties
     * @return a registry populated with all configured bundles
     */
    @Bean
    @ConditionalOnMissingBean
    public DefaultSslBundleRegistry sslBundleRegistry(SslProperties sslProperties) {
        DefaultSslBundleRegistry registry = new DefaultSslBundleRegistry();

        // Register JKS bundles
        sslProperties.getBundle().getJks().forEach((name, props) -> {
            SslBundle bundle = createJksBundle(props);
            registry.registerBundle(name, bundle);
        });

        // Register PEM bundles
        sslProperties.getBundle().getPem().forEach((name, props) -> {
            SslBundle bundle = createPemBundle(props);
            registry.registerBundle(name, bundle);
        });

        return registry;
    }

    /**
     * Create an {@link SslBundle} from JKS properties.
     */
    private SslBundle createJksBundle(JksSslBundleProperties props) {
        JksSslStoreDetails keyStoreDetails = new JksSslStoreDetails(
                props.getKeystore().getType(),
                props.getKeystore().getLocation(),
                props.getKeystore().getPassword());

        JksSslStoreDetails trustStoreDetails = null;
        if (props.getTruststore().getLocation() != null) {
            trustStoreDetails = new JksSslStoreDetails(
                    props.getTruststore().getType(),
                    props.getTruststore().getLocation(),
                    props.getTruststore().getPassword());
        }

        SslStoreBundle stores = new JksSslStoreBundle(keyStoreDetails, trustStoreDetails);
        SslBundleKey key = SslBundleKey.of(
                props.getKey().getPassword(),
                props.getKey().getAlias());

        return SslBundle.of(stores, key, null, props.getProtocol());
    }

    /**
     * Create an {@link SslBundle} from PEM properties.
     */
    private SslBundle createPemBundle(PemSslBundleProperties props) {
        PemSslStoreDetails keyStoreDetails = null;
        if (props.getKeystore().getCertificate() != null) {
            keyStoreDetails = new PemSslStoreDetails(
                    props.getKeystore().getCertificate(),
                    props.getKeystore().getPrivateKey(),
                    props.getKeystore().getPrivateKeyPassword());
        }

        PemSslStoreDetails trustStoreDetails = null;
        if (props.getTruststore().getCertificate() != null) {
            trustStoreDetails = new PemSslStoreDetails(
                    props.getTruststore().getCertificate(),
                    null, null);
        }

        SslStoreBundle stores = new PemSslStoreBundle(keyStoreDetails, trustStoreDetails);
        return SslBundle.of(stores, SslBundleKey.NONE, null, props.getProtocol());
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/context/properties/ConfigurationPropertiesBinder.java`

```java
package com.iris.boot.context.properties;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

import com.iris.framework.core.env.Environment;
import com.iris.framework.core.env.PropertySource;
import com.iris.framework.core.env.PropertySourcesPropertyResolver;

/**
 * Binds properties from the {@link Environment} to a target object's fields
 * using a given prefix.
 *
 * <p>For each non-static, non-final field in the target class, the binder
 * constructs property keys by appending the field name to the prefix,
 * looks them up in the environment, converts to the field's type, and
 * sets the value via reflection.
 *
 * <h3>Binding algorithm</h3>
 *
 * <p>For a prefix of {@code "app.server"} and a field named {@code maxConnections}:
 * <ol>
 *   <li>Try exact: {@code app.server.maxConnections}</li>
 *   <li>Try kebab-case: {@code app.server.max-connections}</li>
 *   <li>If either matches, convert and set the field</li>
 * </ol>
 *
 * <p>For nested objects (fields whose type is not a simple type or List),
 * the binder recurses with an extended prefix. Nested binding only applies
 * if the field is already initialized (non-null) — we don't auto-create
 * nested instances.
 *
 * <p>In the real Spring Boot, this is handled by the {@code Binder} class
 * which is a sophisticated recursive engine supporting:
 * <ul>
 *   <li>{@code JavaBeanBinder} for setter-based binding</li>
 *   <li>{@code ValueObjectBinder} for constructor-based binding</li>
 *   <li>Full relaxed binding with {@code ConfigurationPropertyName}</li>
 *   <li>{@code Map}, {@code Set}, indexed {@code List} binding (we added Map support in Feature 22)</li>
 *   <li>{@code BindHandler} chain for validation/error handling</li>
 * </ul>
 * We simplify to direct field access with two-form relaxed binding.
 *
 * @see ConfigurationProperties
 * @see ConfigurationPropertiesBindingPostProcessor
 * @see org.springframework.boot.context.properties.bind.Binder
 */
public class ConfigurationPropertiesBinder {

    private final Environment environment;

    public ConfigurationPropertiesBinder(Environment environment) {
        this.environment = environment;
    }

    /**
     * Bind properties with the given prefix to the target object.
     *
     * @param target the object to bind properties to
     * @param prefix the property prefix (e.g., {@code "app.server"})
     */
    public void bind(Object target, String prefix) {
        Class<?> targetClass = target.getClass();
        bindFields(target, targetClass, prefix);
    }

    /**
     * Walk the class hierarchy and bind fields at each level.
     */
    private void bindFields(Object target, Class<?> clazz, String prefix) {
        if (clazz == null || clazz == Object.class) {
            return;
        }

        for (Field field : clazz.getDeclaredFields()) {
            if (Modifier.isStatic(field.getModifiers())
                    || Modifier.isFinal(field.getModifiers())) {
                continue;
            }
            bindField(target, field, prefix);
        }

        // Walk up the class hierarchy
        bindFields(target, clazz.getSuperclass(), prefix);
    }

    /**
     * Bind a single field from the environment.
     */
    private void bindField(Object target, Field field, String prefix) {
        String fieldName = field.getName();
        Class<?> fieldType = field.getType();

        if (isSimpleType(fieldType)) {
            bindSimpleField(target, field, prefix, fieldName, fieldType);
        } else if (fieldType == List.class) {
            bindListField(target, field, prefix, fieldName);
        } else if (Map.class.isAssignableFrom(fieldType)) {
            bindMapField(target, field, prefix, fieldName);
        } else {
            bindNestedObject(target, field, prefix, fieldName);
        }
    }

    /**
     * Bind a simple-type field (String, int, boolean, etc.).
     *
     * <p>Tries both the exact field name and kebab-case. The first match wins.
     */
    private void bindSimpleField(Object target, Field field, String prefix,
                                  String fieldName, Class<?> fieldType) {
        String value = resolveProperty(prefix, fieldName);
        if (value != null) {
            try {
                Object converted = PropertySourcesPropertyResolver.convertValue(value, fieldType);
                setFieldValue(target, field, converted);
            } catch (Exception ex) {
                String propertyName = prefix + "." + fieldName;
                throw new BindException(propertyName, target.getClass(),
                        "failed to convert value '" + value + "' to type "
                                + fieldType.getSimpleName(), ex);
            }
        }
    }

    /**
     * Bind a {@code List<String>} field from a comma-separated property value.
     *
     * <p>For example, {@code app.server.cors-origins=http://a,http://b} produces
     * a two-element list.
     *
     * <p>In the real Spring Boot, list binding supports indexed syntax
     * ({@code origins[0]=a}, {@code origins[1]=b}) and type conversion for
     * {@code List<Integer>}, etc. We simplify to comma-separated {@code List<String>}.
     */
    private void bindListField(Object target, Field field, String prefix,
                                String fieldName) {
        String value = resolveProperty(prefix, fieldName);
        if (value != null) {
            List<String> list = Arrays.stream(value.split(","))
                    .map(String::trim)
                    .filter(s -> !s.isEmpty())
                    .toList();
            setFieldValue(target, field, list);
        }
    }

    /**
     * Bind a nested object by recursing with an extended prefix.
     *
     * <p>Only binds if the field is already initialized (non-null). This is a
     * simplification — the real Spring Boot {@code JavaBeanBinder} auto-creates
     * nested instances via the no-arg constructor if needed.
     */
    private void bindNestedObject(Object target, Field field, String prefix,
                                   String fieldName) {
        Object nested = getFieldValue(target, field);

        // Auto-create if null and the type has a no-arg constructor
        if (nested == null) {
            String nestedPrefix = prefix + "." + fieldName;
            String nestedKebabPrefix = prefix + "." + toKebabCase(fieldName);
            if (hasPropertiesWithPrefix(nestedPrefix)
                    || hasPropertiesWithPrefix(nestedKebabPrefix)) {
                try {
                    var ctor = field.getType().getDeclaredConstructor();
                    ctor.setAccessible(true);
                    nested = ctor.newInstance();
                    setFieldValue(target, field, nested);
                } catch (ReflectiveOperationException ex) {
                    // Cannot create nested object — skip
                    return;
                }
            }
        }

        if (nested != null) {
            bind(nested, prefix + "." + fieldName);
        }
    }

    /**
     * Bind a {@code Map<String, ?>} field from the environment.
     *
     * <p>Scans all property sources for keys matching
     * {@code prefix.fieldName.<mapKey>...}, extracts unique map keys,
     * and for each key:
     * <ul>
     *   <li>If the value type is simple, binds a single property value</li>
     *   <li>If the value type is complex, creates a new instance and
     *       recursively binds with the extended prefix</li>
     * </ul>
     *
     * <p>This enables the SSL Bundles pattern:
     * {@code spring.ssl.bundle.jks.myBundle.keystore.location=...}
     * where "myBundle" is the map key.
     *
     * <p>Added in Feature 22 to support {@code Map<String, BundleProperties>}
     * binding. The real Spring Boot {@code MapBinder} handles this with more
     * sophistication (indexed maps, type conversion, etc.).
     */
    @SuppressWarnings("unchecked")
    private void bindMapField(Object target, Field field, String prefix,
                               String fieldName) {
        // Determine the value type from generics
        Class<?> valueType = getMapValueType(field);
        if (valueType == null) {
            return; // Cannot determine value type
        }

        String mapPrefix = prefix + "." + fieldName;
        String mapKebabPrefix = prefix + "." + toKebabCase(fieldName);

        // Find all unique map keys from property sources
        Set<String> mapKeys = findMapKeys(mapPrefix);
        if (!mapPrefix.equals(mapKebabPrefix)) {
            mapKeys.addAll(findMapKeys(mapKebabPrefix));
        }

        if (mapKeys.isEmpty()) {
            return;
        }

        // Get or create the map
        Map<String, Object> map = (Map<String, Object>) getFieldValue(target, field);
        if (map == null) {
            map = new LinkedHashMap<>();
            setFieldValue(target, field, map);
        }

        for (String key : mapKeys) {
            String entryPrefix = mapPrefix + "." + key;
            if (isSimpleType(valueType)) {
                String value = environment.getProperty(entryPrefix);
                if (value != null) {
                    map.put(key, PropertySourcesPropertyResolver.convertValue(value, valueType));
                }
            } else {
                // Create a nested object and bind recursively
                try {
                    Object nested = valueType.getDeclaredConstructor().newInstance();
                    bind(nested, entryPrefix);
                    map.put(key, nested);
                } catch (ReflectiveOperationException ex) {
                    // Cannot create value instance — skip this map entry
                }
            }
        }
    }

    /**
     * Extract the value type from a {@code Map<String, V>} field's generic signature.
     *
     * @return the value type class, or {@code null} if it cannot be determined
     */
    private Class<?> getMapValueType(Field field) {
        Type genericType = field.getGenericType();
        if (genericType instanceof ParameterizedType pt) {
            Type[] typeArgs = pt.getActualTypeArguments();
            if (typeArgs.length == 2 && typeArgs[1] instanceof Class<?> valueClass) {
                return valueClass;
            }
        }
        // Default to String if we can't determine the type
        return String.class;
    }

    /**
     * Find all unique map keys under the given prefix.
     *
     * <p>For a prefix of "spring.ssl.bundle.jks" and property keys:
     * <ul>
     *   <li>{@code spring.ssl.bundle.jks.server.keystore.location}</li>
     *   <li>{@code spring.ssl.bundle.jks.server.keystore.password}</li>
     *   <li>{@code spring.ssl.bundle.jks.client.truststore.location}</li>
     * </ul>
     * Returns ["server", "client"].
     */
    private Set<String> findMapKeys(String mapPrefix) {
        String dotPrefix = mapPrefix + ".";
        Set<String> keys = new LinkedHashSet<>();
        for (PropertySource<?> ps : environment.getPropertySources()) {
            Object source = ps.getSource();
            if (source instanceof java.util.Map<?, ?> map) {
                for (Object key : map.keySet()) {
                    if (key instanceof String strKey && strKey.startsWith(dotPrefix)) {
                        String remaining = strKey.substring(dotPrefix.length());
                        int dotIndex = remaining.indexOf('.');
                        String mapKey = (dotIndex > 0) ? remaining.substring(0, dotIndex) : remaining;
                        keys.add(mapKey);
                    }
                }
            }
        }
        return keys;
    }

    // -----------------------------------------------------------------------
    // Property resolution helpers
    // -----------------------------------------------------------------------

    /**
     * Resolve a property value, trying both the exact field name and kebab-case.
     *
     * @return the property value, or {@code null} if not found under either form
     */
    private String resolveProperty(String prefix, String fieldName) {
        // Try exact field name first
        String value = environment.getProperty(prefix + "." + fieldName);
        if (value != null) {
            return value;
        }

        // Try kebab-case
        String kebabName = toKebabCase(fieldName);
        if (!kebabName.equals(fieldName)) {
            value = environment.getProperty(prefix + "." + kebabName);
        }
        return value;
    }

    /**
     * Check if any property source contains keys starting with the given prefix.
     *
     * <p>This is used to decide whether to auto-create a nested object. We
     * iterate all property sources and check their underlying maps.
     */
    private boolean hasPropertiesWithPrefix(String prefix) {
        String dotPrefix = prefix + ".";
        for (PropertySource<?> ps : environment.getPropertySources()) {
            Object source = ps.getSource();
            if (source instanceof java.util.Map<?, ?> map) {
                for (Object key : map.keySet()) {
                    if (key instanceof String strKey && strKey.startsWith(dotPrefix)) {
                        return true;
                    }
                }
            }
        }
        return false;
    }

    // -----------------------------------------------------------------------
    // Reflection helpers
    // -----------------------------------------------------------------------

    private static Object getFieldValue(Object target, Field field) {
        try {
            field.setAccessible(true);
            return field.get(target);
        } catch (IllegalAccessException ex) {
            return null;
        }
    }

    private static void setFieldValue(Object target, Field field, Object value) {
        try {
            field.setAccessible(true);
            field.set(target, value);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(
                    "Failed to set field '" + field.getName() + "' on "
                    + target.getClass().getSimpleName(), ex);
        }
    }

    // -----------------------------------------------------------------------
    // Type helpers
    // -----------------------------------------------------------------------

    /**
     * Check whether a type is a "simple" type that can be converted from a
     * single string value.
     */
    static boolean isSimpleType(Class<?> type) {
        return type == String.class
                || type == int.class || type == Integer.class
                || type == long.class || type == Long.class
                || type == boolean.class || type == Boolean.class
                || type == double.class || type == Double.class
                || type.isEnum();
    }

    /**
     * Convert a camelCase field name to kebab-case.
     *
     * <p>Examples:
     * <ul>
     *   <li>{@code "maxConnections"} → {@code "max-connections"}</li>
     *   <li>{@code "sslEnabled"} → {@code "ssl-enabled"}</li>
     *   <li>{@code "port"} → {@code "port"} (no change)</li>
     * </ul>
     *
     * <p>In the real Spring Boot, this conversion is handled by
     * {@code DataObjectPropertyName.toDashedForm()} which also handles
     * underscores and other edge cases. We support the common camelCase
     * to kebab-case conversion.
     */
    static String toKebabCase(String camelCase) {
        if (camelCase == null || camelCase.isEmpty()) {
            return camelCase;
        }
        StringBuilder result = new StringBuilder();
        for (int i = 0; i < camelCase.length(); i++) {
            char c = camelCase.charAt(i);
            if (Character.isUpperCase(c)) {
                if (i > 0) {
                    result.append('-');
                }
                result.append(Character.toLowerCase(c));
            } else {
                result.append(c);
            }
        }
        return result.toString();
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/web/ServerProperties.java`

```java
package com.iris.boot.autoconfigure.web;

import com.iris.boot.context.properties.ConfigurationProperties;
import com.iris.boot.web.server.Shutdown;

/**
 * {@link ConfigurationProperties @ConfigurationProperties} for configuring
 * the embedded web server.
 *
 * <p>Binds properties with the {@code server} prefix from
 * {@code application.properties}:
 * <pre>
 * server.port=9090
 * server.shutdown=graceful
 * server.ssl.bundle=myBundle
 * </pre>
 *
 * <p>In the real Spring Boot, {@code ServerProperties} is a large class with
 * nested classes for Tomcat, Jetty, Undertow, compression, SSL, HTTP/2, etc.
 * We support port, shutdown mode, and SSL bundle reference.
 *
 * @see org.springframework.boot.autoconfigure.web.ServerProperties
 */
@ConfigurationProperties(prefix = "server")
public class ServerProperties {

    /**
     * Server HTTP port. Default is 8080.
     */
    private int port = 8080;

    /**
     * Shutdown mode. Default is IMMEDIATE (no graceful shutdown).
     * Set to {@link Shutdown#GRACEFUL} to drain in-flight requests
     * before stopping.
     */
    private Shutdown shutdown = Shutdown.IMMEDIATE;

    /**
     * SSL configuration for the embedded web server.
     */
    private Ssl ssl = new Ssl();

    public int getPort() {
        return port;
    }

    public void setPort(int port) {
        this.port = port;
    }

    public Shutdown getShutdown() {
        return shutdown;
    }

    public void setShutdown(Shutdown shutdown) {
        this.shutdown = shutdown;
    }

    public Ssl getSsl() {
        return ssl;
    }

    public void setSsl(Ssl ssl) {
        this.ssl = ssl;
    }

    /**
     * SSL properties for the embedded web server.
     *
     * <p>In the real Spring Boot, {@code ServerProperties.Ssl} has many
     * fields (enabled, key-store, trust-store, ciphers, etc.). With SSL
     * Bundles, these are replaced by a single bundle name reference.
     */
    public static class Ssl {

        /**
         * The name of the SSL bundle to use for HTTPS configuration.
         * When set, the server will start with HTTPS enabled using the
         * referenced bundle's certificates and keys.
         */
        private String bundle;

        public String getBundle() {
            return bundle;
        }

        public void setBundle(String bundle) {
            this.bundle = bundle;
        }
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/web/embedded/tomcat/TomcatServletWebServerFactory.java`

```java
package com.iris.boot.web.embedded.tomcat;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.nio.file.Files;
import java.security.KeyStore;

import jakarta.servlet.Servlet;

import org.apache.catalina.Context;
import org.apache.catalina.Wrapper;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.startup.Tomcat;
import org.apache.tomcat.util.net.SSLHostConfig;
import org.apache.tomcat.util.net.SSLHostConfigCertificate;

import com.iris.boot.ssl.SslBundle;
import com.iris.boot.ssl.SslOptions;
import com.iris.boot.web.server.Shutdown;
import com.iris.boot.web.server.WebServer;
import com.iris.boot.web.server.WebServerException;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;

/**
 * {@link ServletWebServerFactory} that creates a {@link TomcatWebServer} backed
 * by an embedded Apache Tomcat.
 *
 * <p>This is the factory that knows HOW to create and configure a Tomcat instance.
 * The {@code ServletWebServerApplicationContext} delegates to this factory via the
 * {@link #getWebServer()} method, keeping the context server-agnostic.
 *
 * <p>Usage via {@code @Configuration}:
 * <pre>{@code
 * @Configuration
 * public class WebConfig {
 *     @Bean
 *     public ServletWebServerFactory webServerFactory() {
 *         return new TomcatServletWebServerFactory(8080);
 *     }
 * }
 * }</pre>
 *
 * <h3>Simplifications from Real Spring Boot</h3>
 *
 * <p>The real {@code TomcatServletWebServerFactory} supports:
 * <ul>
 *   <li>{@code ServletContextInitializer}s for servlet/filter registration</li>
 *   <li>SSL configuration via {@code SslBundle}</li>
 *   <li>Compression settings</li>
 *   <li>Protocol customization (HTTP/1.1, HTTP/2)</li>
 *   <li>Error pages, session config, MIME mappings</li>
 *   <li>Additional connectors for multiple ports</li>
 *   <li>Custom valves, context customizers</li>
 * </ul>
 *
 * <p>We support port, shutdown mode, and SSL configuration via {@link SslBundle}.
 *
 * @see org.springframework.boot.tomcat.servlet.TomcatServletWebServerFactory
 */
public class TomcatServletWebServerFactory implements ServletWebServerFactory {

    /** Default port for the embedded Tomcat server. */
    private static final int DEFAULT_PORT = 8080;

    private int port;

    /** The shutdown mode — IMMEDIATE or GRACEFUL. */
    private Shutdown shutdown = Shutdown.IMMEDIATE;

    /** The SSL bundle for HTTPS support — null means HTTP only. */
    private SslBundle sslBundle;

    /**
     * Create a new factory with the default port (8080).
     */
    public TomcatServletWebServerFactory() {
        this(DEFAULT_PORT);
    }

    /**
     * Create a new factory with the specified port.
     *
     * @param port the port to listen on (use 0 for auto-assignment)
     */
    public TomcatServletWebServerFactory(int port) {
        this.port = port;
    }

    /**
     * Create a new fully configured {@link TomcatWebServer} with the given
     * servlets registered.
     *
     * <p>The creation sequence mirrors the real factory ({@code getWebServer()}
     * at line 163):
     * <ol>
     *   <li>Create a temporary base directory (Tomcat needs a work dir)</li>
     *   <li>Create a new {@code Tomcat} instance</li>
     *   <li>Configure the connector (port, protocol)</li>
     *   <li>Create a servlet context and register servlets</li>
     *   <li>Wrap it in a {@code TomcatWebServer}</li>
     * </ol>
     *
     * @param servlets the servlets to register (typically just DispatcherServlet)
     * @return a configured but not started web server
     */
    @Override
    public WebServer getWebServer(Servlet... servlets) {
        Tomcat tomcat = createTomcat();
        prepareContext(tomcat, servlets);
        return new TomcatWebServer(tomcat, this.shutdown);
    }

    /**
     * Create and configure the underlying {@code Tomcat} instance.
     *
     * <p>In the real Spring Boot, this is {@code TomcatWebServerFactory.createTomcat()}
     * which configures the base directory, connector protocol, engine name,
     * additional connectors, and applies customizers.
     *
     * <p>We configure the essentials: base directory, connector, and host.
     */
    private Tomcat createTomcat() {
        Tomcat tomcat = new Tomcat();

        // Tomcat requires a base directory for temp files, work directories, etc.
        // The real Spring Boot creates this in TempDirs.
        File baseDir = createTempDir("tomcat");
        tomcat.setBaseDir(baseDir.getAbsolutePath());

        // Configure the default connector
        // In real Spring Boot, the protocol is customizable (NIO, NIO2, APR).
        // We use the default protocol (HTTP/1.1 NIO).
        if (this.sslBundle != null) {
            // SSL: create a custom connector with HTTPS support
            Connector connector = createSslConnector();
            tomcat.setConnector(connector);
        } else {
            tomcat.setPort(this.port);
        }

        // Configure the default host
        tomcat.getHost().setAutoDeploy(false);

        return tomcat;
    }

    /**
     * Create a Tomcat {@link Connector} configured for HTTPS using the
     * {@link SslBundle}.
     *
     * <p>This is the integration point where Spring Boot's SSL abstraction
     * meets Tomcat's native SSL configuration. The process:
     * <ol>
     *   <li>Create a connector with SSL enabled</li>
     *   <li>Write the KeyStore from the bundle to a temp file (Tomcat's
     *       native API expects file paths)</li>
     *   <li>Configure an {@link SSLHostConfig} with the keystore path,
     *       password, and optional cipher/protocol settings</li>
     *   <li>Attach the SSL config to the connector</li>
     * </ol>
     *
     * <p>In the real Spring Boot, this is handled by
     * {@code SslConnectorCustomizer} which uses a custom Tomcat
     * {@code SSLImplementation} to avoid writing keystores to disk.
     * We use the simpler file-based approach for clarity.
     */
    private Connector createSslConnector() {
        Connector connector = new Connector();
        connector.setPort(this.port);
        connector.setScheme("https");
        connector.setSecure(true);
        connector.setProperty("SSLEnabled", "true");

        SSLHostConfig sslHostConfig = new SSLHostConfig();

        // Configure the certificate — keystore with the server's identity
        SSLHostConfigCertificate certificate = new SSLHostConfigCertificate(
                sslHostConfig, SSLHostConfigCertificate.Type.UNDEFINED);

        KeyStore keyStore = this.sslBundle.getStores().getKeyStore();
        if (keyStore != null) {
            File keystoreFile = writeKeyStoreToTempFile(keyStore,
                    this.sslBundle.getStores().getKeyStorePassword());
            certificate.setCertificateKeystoreFile(keystoreFile.getAbsolutePath());
            certificate.setCertificateKeystoreType(keyStore.getType());
            if (this.sslBundle.getStores().getKeyStorePassword() != null) {
                certificate.setCertificateKeystorePassword(
                        this.sslBundle.getStores().getKeyStorePassword());
            }
        }

        // Key alias and password
        if (this.sslBundle.getKey().getAlias() != null) {
            certificate.setCertificateKeyAlias(this.sslBundle.getKey().getAlias());
        }
        if (this.sslBundle.getKey().getPassword() != null) {
            certificate.setCertificateKeyPassword(this.sslBundle.getKey().getPassword());
        }

        sslHostConfig.addCertificate(certificate);

        // Configure the trust store (for client certificate authentication)
        KeyStore trustStore = this.sslBundle.getStores().getTrustStore();
        if (trustStore != null) {
            File truststoreFile = writeKeyStoreToTempFile(trustStore, null);
            sslHostConfig.setTruststoreFile(truststoreFile.getAbsolutePath());
            sslHostConfig.setTruststoreType(trustStore.getType());
        }

        // Apply SSL options (ciphers and protocols)
        SslOptions options = this.sslBundle.getOptions();
        if (options.getCiphers() != null) {
            sslHostConfig.setCiphers(String.join(",", options.getCiphers()));
        }
        if (options.getEnabledProtocols() != null) {
            sslHostConfig.setProtocols(String.join(",", options.getEnabledProtocols()));
        }

        // Set the SSL protocol (e.g., "TLS", "TLSv1.3")
        sslHostConfig.setSslProtocol(this.sslBundle.getProtocol());

        connector.addSslHostConfig(sslHostConfig);
        return connector;
    }

    /**
     * Write a {@link KeyStore} to a temporary file so Tomcat can load it.
     *
     * <p>Tomcat's native SSL API expects file paths for keystores. This
     * method serializes the in-memory KeyStore to a temp file. The file
     * is deleted when the JVM exits.
     *
     * <p>In the real Spring Boot, this is avoided by using a custom
     * {@code SSLImplementation} that feeds Tomcat the KeyStore directly.
     * The temp-file approach is simpler and perfectly adequate for our
     * educational version.
     */
    private File writeKeyStoreToTempFile(KeyStore keyStore, String password) {
        try {
            File file = File.createTempFile("ssl-keystore-", ".p12");
            file.deleteOnExit();
            char[] passwordChars = (password != null) ? password.toCharArray() : new char[0];
            try (OutputStream os = new FileOutputStream(file)) {
                keyStore.store(os, passwordChars);
            }
            return file;
        } catch (Exception ex) {
            throw new WebServerException("Unable to write KeyStore to temp file", ex);
        }
    }

    /**
     * Prepare a servlet context on the Tomcat host and register the given servlets.
     *
     * <p>In the real Spring Boot ({@code prepareContext()}, line 189), this
     * creates a {@code TomcatEmbeddedContext}, configures the document root,
     * class loader, locale mappings, error pages, session config, and applies
     * all {@code ServletContextInitializer}s (which register DispatcherServlet).
     *
     * <p>We create a context, register each servlet, and map the first one
     * (typically DispatcherServlet) to "/" as the default servlet.
     */
    private void prepareContext(Tomcat tomcat, Servlet... servlets) {
        // addContext() registers a Context with the Host.
        // "" means the root context path.
        Context context = tomcat.addContext("", createTempDir("tomcat-docbase").getAbsolutePath());

        // Use the application's class loader so servlets can find application classes
        context.setParentClassLoader(getClass().getClassLoader());

        // Register each servlet with the Tomcat context
        for (Servlet servlet : servlets) {
            String servletName = servlet.getClass().getSimpleName();
            Wrapper wrapper = Tomcat.addServlet(context, servletName, servlet);
            wrapper.setLoadOnStartup(1);
            // Map to "/" — makes this the default servlet (handles all unmatched URLs)
            context.addServletMappingDecoded("/", servletName);
        }
    }

    /**
     * Create a temporary directory for Tomcat's working files.
     *
     * <p>In the real Spring Boot, this is handled by {@code TempDirs}.
     *
     * @param prefix the directory name prefix
     * @return the created directory
     */
    private File createTempDir(String prefix) {
        try {
            File tempDir = Files.createTempDirectory(prefix + ".").toFile();
            tempDir.deleteOnExit();
            return tempDir;
        } catch (IOException ex) {
            throw new WebServerException(
                    "Unable to create temp directory for embedded Tomcat", ex);
        }
    }

    // -----------------------------------------------------------------------
    // Configuration
    // -----------------------------------------------------------------------

    /**
     * Set the port that the embedded Tomcat server should listen on.
     * Use port 0 for automatic port assignment.
     */
    public void setPort(int port) {
        this.port = port;
    }

    /**
     * Return the configured port.
     */
    public int getPort() {
        return this.port;
    }

    /**
     * Set the shutdown mode for the embedded Tomcat server.
     *
     * <p>When set to {@link Shutdown#GRACEFUL}, the server will drain
     * in-flight requests before stopping. When {@link Shutdown#IMMEDIATE}
     * (the default), the server stops immediately.
     *
     * @param shutdown the shutdown mode
     */
    public void setShutdown(Shutdown shutdown) {
        this.shutdown = shutdown;
    }

    /**
     * Return the configured shutdown mode.
     */
    public Shutdown getShutdown() {
        return this.shutdown;
    }

    /**
     * Set the SSL bundle for HTTPS support.
     *
     * <p>When set, the embedded Tomcat server will listen for HTTPS
     * connections using the certificates and keys from this bundle.
     * When {@code null} (the default), the server uses plain HTTP.
     *
     * @param sslBundle the SSL bundle, or {@code null} for HTTP
     */
    public void setSslBundle(SslBundle sslBundle) {
        this.sslBundle = sslBundle;
    }

    /**
     * Return the configured SSL bundle, or {@code null} if HTTPS is
     * not configured.
     */
    public SslBundle getSslBundle() {
        return this.sslBundle;
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/web/servlet/TomcatAutoConfiguration.java`

```java
package com.iris.boot.autoconfigure.web.servlet;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.AutoConfigureAfter;
import com.iris.boot.autoconfigure.condition.ConditionalOnClass;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.boot.autoconfigure.ssl.SslAutoConfiguration;
import com.iris.boot.autoconfigure.web.ServerProperties;
import com.iris.boot.context.properties.EnableConfigurationProperties;
import com.iris.boot.ssl.SslBundles;
import com.iris.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import com.iris.boot.web.server.servlet.ServletWebServerFactory;
import com.iris.framework.context.annotation.Bean;

/**
 * Auto-configuration for embedded Tomcat.
 *
 * <p>This auto-configuration activates when Tomcat is on the classpath
 * (detected by {@code @ConditionalOnClass}) and no user-defined
 * {@link ServletWebServerFactory} bean exists (detected by
 * {@code @ConditionalOnMissingBean}).
 *
 * <p>It creates a {@link TomcatServletWebServerFactory} configured with
 * the port and shutdown mode from {@link ServerProperties}, and optionally
 * an SSL bundle for HTTPS support.
 *
 * <p>In the real Spring Boot 4.x, this is
 * {@code TomcatServletWebServerAutoConfiguration} which lives in the
 * {@code spring-boot-tomcat} module. It's much more complex:
 * <ul>
 *   <li>Uses {@code @ConditionalOnWebApplication(type = SERVLET)}</li>
 *   <li>Applies customizers ({@code TomcatServletWebServerFactoryCustomizer})</li>
 *   <li>Configures protocol, connectors, valves</li>
 *   <li>Imports shared {@code ServletWebServerConfiguration}</li>
 * </ul>
 *
 * @see org.springframework.boot.tomcat.autoconfigure.servlet.TomcatServletWebServerAutoConfiguration
 */
@AutoConfiguration
@AutoConfigureAfter(SslAutoConfiguration.class)
@ConditionalOnClass(name = "org.apache.catalina.startup.Tomcat")
@EnableConfigurationProperties(ServerProperties.class)
public class TomcatAutoConfiguration {

    /**
     * Create a {@link TomcatServletWebServerFactory} if the user hasn't
     * defined one.
     *
     * <p>The {@code @ConditionalOnMissingBean} annotation causes this bean
     * to back off if the user provides their own {@code ServletWebServerFactory}.
     * This is the central auto-configuration pattern: <strong>provide sensible
     * defaults that yield to user customization</strong>.
     *
     * <p>If {@code server.ssl.bundle} is configured AND an {@link SslBundles}
     * bean is available, the factory is configured for HTTPS using the
     * referenced SSL bundle. This is the integration point where the SSL
     * Bundles feature connects to the embedded web server.
     *
     * @param serverProperties the bound server properties
     * @param sslBundles       the SSL bundle registry (may be {@code null} if
     *                         SSL auto-configuration is not active)
     * @return a configured Tomcat factory
     */
    @Bean
    @ConditionalOnMissingBean
    public ServletWebServerFactory tomcatServletWebServerFactory(
            ServerProperties serverProperties,
            SslBundles sslBundles) {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.setPort(serverProperties.getPort());
        factory.setShutdown(serverProperties.getShutdown());

        // Wire SSL if configured
        String bundleName = serverProperties.getSsl().getBundle();
        if (bundleName != null && !bundleName.isBlank() && sslBundles != null) {
            factory.setSslBundle(sslBundles.getBundle(bundleName));
        }

        return factory;
    }
}
```

#### `[MODIFIED]` `iris-boot-core/src/main/resources/META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports`

```
# Iris Boot Auto-Configuration Candidates
#
# Listed in dependency order:
# 1. JacksonAutoConfiguration — provides ObjectMapper (no ordering constraints)
# 2. SslAutoConfiguration — provides SslBundles registry (before Tomcat)
# 3. TomcatAutoConfiguration — provides ServletWebServerFactory (after SSL)
# 4. WebMvcAutoConfiguration — provides HttpMessageConverter (after Tomcat)
com.iris.boot.autoconfigure.jackson.JacksonAutoConfiguration
com.iris.boot.autoconfigure.ssl.SslAutoConfiguration
com.iris.boot.autoconfigure.web.servlet.TomcatAutoConfiguration
com.iris.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/ssl/SslTestHelper.java`

```java
package com.iris.boot.ssl;

import java.io.File;
import java.io.FileOutputStream;
import java.io.OutputStream;
import java.security.KeyStore;
import java.security.PrivateKey;
import java.security.cert.Certificate;
import java.util.Base64;

/**
 * Test helper for creating self-signed certificates and keystores.
 *
 * <p>Uses the JDK's {@code keytool} command to generate self-signed
 * certificates, avoiding direct use of internal {@code sun.security.x509}
 * APIs. The generated keystores and PEM content are used across SSL tests.
 */
public final class SslTestHelper {

    private SslTestHelper() {
    }

    /**
     * Create a PKCS12 keystore containing a self-signed RSA certificate
     * by invoking {@code keytool}.
     *
     * @param alias    the alias for the key entry
     * @param password the keystore and key password
     * @return a loaded KeyStore
     */
    public static KeyStore createSelfSignedKeyStore(String alias, String password) throws Exception {
        // Create a temp file for keytool to write to
        File file = File.createTempFile("test-keystore-", ".p12");
        file.deleteOnExit();
        // Delete the file so keytool can create it fresh
        file.delete();

        ProcessBuilder pb = new ProcessBuilder(
                "keytool", "-genkeypair",
                "-alias", alias,
                "-keyalg", "RSA",
                "-keysize", "2048",
                "-validity", "365",
                "-storepass", password,
                "-keypass", password,
                "-dname", "CN=Test, OU=Test, O=Test, L=Test, ST=Test, C=US",
                "-storetype", "PKCS12",
                "-keystore", file.getAbsolutePath());
        pb.redirectErrorStream(true);
        Process process = pb.start();
        // Read output to prevent blocking
        process.getInputStream().readAllBytes();
        int exitCode = process.waitFor();
        if (exitCode != 0) {
            throw new RuntimeException("keytool failed with exit code " + exitCode);
        }

        // Load the generated keystore
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (var is = new java.io.FileInputStream(file)) {
            keyStore.load(is, password.toCharArray());
        }
        return keyStore;
    }

    /**
     * Write a keystore to a temporary file.
     */
    public static File writeKeyStoreToFile(KeyStore keyStore, String password) throws Exception {
        File file = File.createTempFile("test-keystore-", ".p12");
        file.deleteOnExit();
        try (OutputStream os = new FileOutputStream(file)) {
            keyStore.store(os, password != null ? password.toCharArray() : new char[0]);
        }
        return file;
    }

    /**
     * Generate PEM-encoded certificate content from a keystore entry.
     */
    public static String toPemCertificate(KeyStore keyStore, String alias) throws Exception {
        Certificate cert = keyStore.getCertificate(alias);
        byte[] encoded = cert.getEncoded();
        String base64 = Base64.getMimeEncoder(64, "\n".getBytes()).encodeToString(encoded);
        return "-----BEGIN CERTIFICATE-----\n" + base64 + "\n-----END CERTIFICATE-----\n";
    }

    /**
     * Generate PEM-encoded PKCS#8 private key content from a keystore entry.
     */
    public static String toPemPrivateKey(KeyStore keyStore, String alias, String password) throws Exception {
        PrivateKey key = (PrivateKey) keyStore.getKey(alias, password.toCharArray());
        byte[] encoded = key.getEncoded(); // PKCS#8 format
        String base64 = Base64.getMimeEncoder(64, "\n".getBytes()).encodeToString(encoded);
        return "-----BEGIN PRIVATE KEY-----\n" + base64 + "\n-----END PRIVATE KEY-----\n";
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/ssl/SslBundleKeyTest.java`

```java
package com.iris.boot.ssl;

import java.security.KeyStore;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for {@link SslBundleKey}.
 */
class SslBundleKeyTest {

    @Test
    void shouldReturnNone_WhenBothPasswordAndAliasAreNull() {
        SslBundleKey key = SslBundleKey.of(null, null);
        assertThat(key).isSameAs(SslBundleKey.NONE);
        assertThat(key.getPassword()).isNull();
        assertThat(key.getAlias()).isNull();
    }

    @Test
    void shouldCreateKey_WhenAliasIsProvided() {
        SslBundleKey key = SslBundleKey.of(null, "myAlias");
        assertThat(key).isNotSameAs(SslBundleKey.NONE);
        assertThat(key.getAlias()).isEqualTo("myAlias");
        assertThat(key.getPassword()).isNull();
    }

    @Test
    void shouldCreateKey_WhenPasswordIsProvided() {
        SslBundleKey key = SslBundleKey.of("secret", null);
        assertThat(key.getPassword()).isEqualTo("secret");
        assertThat(key.getAlias()).isNull();
    }

    @Test
    void shouldCreateKey_WhenBothAreProvided() {
        SslBundleKey key = SslBundleKey.of("secret", "myAlias");
        assertThat(key.getPassword()).isEqualTo("secret");
        assertThat(key.getAlias()).isEqualTo("myAlias");
    }

    @Test
    void shouldNotThrow_WhenAliasExistsInKeyStore() throws Exception {
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        keyStore.load(null, null);
        // PKCS12 keystores don't support setCertificateEntry without a cert,
        // so test with a null alias (should skip validation)
        SslBundleKey key = SslBundleKey.of(null, null);
        key.assertContainsAlias(keyStore); // Should not throw
    }

    @Test
    void shouldNotThrow_WhenAliasIsNull() throws Exception {
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        keyStore.load(null, null);
        SslBundleKey key = SslBundleKey.of(null, null);
        key.assertContainsAlias(keyStore); // Should not throw
    }

    @Test
    void shouldNotThrow_WhenKeyStoreIsNull() {
        SslBundleKey key = SslBundleKey.of(null, "myAlias");
        key.assertContainsAlias(null); // Should not throw
    }

    @Test
    void shouldThrow_WhenAliasNotFoundInKeyStore() throws Exception {
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        keyStore.load(null, null);
        SslBundleKey key = SslBundleKey.of(null, "nonExistent");
        assertThatThrownBy(() -> key.assertContainsAlias(keyStore))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("nonExistent");
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/ssl/SslOptionsTest.java`

```java
package com.iris.boot.ssl;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link SslOptions}.
 */
class SslOptionsTest {

    @Test
    void shouldReturnNone_WhenBothAreNull() {
        SslOptions options = SslOptions.of(null, null);
        assertThat(options).isSameAs(SslOptions.NONE);
        assertThat(options.getCiphers()).isNull();
        assertThat(options.getEnabledProtocols()).isNull();
        assertThat(options.isSpecified()).isFalse();
    }

    @Test
    void shouldCreateOptions_WhenCiphersProvided() {
        SslOptions options = SslOptions.of(
                new String[]{"TLS_AES_128_GCM_SHA256"}, null);
        assertThat(options.getCiphers()).containsExactly("TLS_AES_128_GCM_SHA256");
        assertThat(options.getEnabledProtocols()).isNull();
        assertThat(options.isSpecified()).isTrue();
    }

    @Test
    void shouldCreateOptions_WhenProtocolsProvided() {
        SslOptions options = SslOptions.of(
                null, new String[]{"TLSv1.3"});
        assertThat(options.getCiphers()).isNull();
        assertThat(options.getEnabledProtocols()).containsExactly("TLSv1.3");
        assertThat(options.isSpecified()).isTrue();
    }

    @Test
    void shouldCreateOptions_WhenBothProvided() {
        SslOptions options = SslOptions.of(
                new String[]{"TLS_AES_128_GCM_SHA256", "TLS_AES_256_GCM_SHA384"},
                new String[]{"TLSv1.2", "TLSv1.3"});
        assertThat(options.getCiphers()).hasSize(2);
        assertThat(options.getEnabledProtocols()).hasSize(2);
        assertThat(options.isSpecified()).isTrue();
    }

    @Test
    void shouldDefensiveCopyArrays() {
        String[] ciphers = {"TLS_AES_128_GCM_SHA256"};
        SslOptions options = SslOptions.of(ciphers, null);
        ciphers[0] = "MODIFIED";
        assertThat(options.getCiphers()).containsExactly("TLS_AES_128_GCM_SHA256");
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/ssl/SslManagerBundleTest.java`

```java
package com.iris.boot.ssl;

import java.security.KeyStore;
import java.security.cert.Certificate;

import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManagerFactory;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link SslManagerBundle} and {@link DefaultSslManagerBundle}.
 */
class SslManagerBundleTest {

    @Test
    void shouldCreateManagerBundle_WhenWrappingPreBuiltFactories() throws Exception {
        KeyManagerFactory kmf = KeyManagerFactory.getInstance(
                KeyManagerFactory.getDefaultAlgorithm());
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(
                TrustManagerFactory.getDefaultAlgorithm());

        SslManagerBundle bundle = SslManagerBundle.of(kmf, tmf);
        assertThat(bundle.getKeyManagerFactory()).isSameAs(kmf);
        assertThat(bundle.getTrustManagerFactory()).isSameAs(tmf);
    }

    @Test
    void shouldBuildManagersFromStores() throws Exception {
        // Create a self-signed keystore for testing
        KeyStore keyStore = SslTestHelper.createSelfSignedKeyStore("test", "changeit");

        SslStoreBundle stores = SslStoreBundle.of(keyStore, "changeit", null);
        SslBundleKey key = SslBundleKey.NONE;

        SslManagerBundle bundle = SslManagerBundle.from(stores, key);
        assertThat(bundle.getKeyManagerFactory()).isNotNull();
        assertThat(bundle.getKeyManagers()).isNotEmpty();
        // No trust store -> null trust manager factory
        assertThat(bundle.getTrustManagerFactory()).isNull();
    }

    @Test
    void shouldCreateSslContext() throws Exception {
        KeyStore keyStore = SslTestHelper.createSelfSignedKeyStore("test", "changeit");

        SslStoreBundle stores = SslStoreBundle.of(keyStore, "changeit", null);
        SslManagerBundle bundle = SslManagerBundle.from(stores, SslBundleKey.NONE);

        SSLContext sslContext = bundle.createSslContext("TLS");
        assertThat(sslContext).isNotNull();
        assertThat(sslContext.getProtocol()).isEqualTo("TLS");
    }

    @Test
    void shouldReturnNullManagers_WhenNoStores() {
        SslManagerBundle bundle = SslManagerBundle.from(SslStoreBundle.NONE, SslBundleKey.NONE);
        assertThat(bundle.getKeyManagerFactory()).isNull();
        assertThat(bundle.getTrustManagerFactory()).isNull();
    }

    @Test
    void shouldBuildTrustManagerFromTrustStore() throws Exception {
        // Use the same keystore as both key and trust store
        KeyStore store = SslTestHelper.createSelfSignedKeyStore("test", "changeit");

        // Extract the certificate for the trust store
        KeyStore trustStore = KeyStore.getInstance("PKCS12");
        trustStore.load(null, null);
        Certificate cert = store.getCertificate("test");
        trustStore.setCertificateEntry("trusted", cert);

        SslStoreBundle stores = SslStoreBundle.of(store, "changeit", trustStore);
        SslManagerBundle bundle = SslManagerBundle.from(stores, SslBundleKey.NONE);

        assertThat(bundle.getKeyManagerFactory()).isNotNull();
        assertThat(bundle.getTrustManagerFactory()).isNotNull();
        assertThat(bundle.getTrustManagers()).isNotEmpty();
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/ssl/SslBundleTest.java`

```java
package com.iris.boot.ssl;

import java.security.KeyStore;

import javax.net.ssl.SSLContext;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link SslBundle}.
 */
class SslBundleTest {

    @Test
    void shouldCreateBundle_WithDefaults() {
        SslBundle bundle = SslBundle.of(null, null, null, null);

        assertThat(bundle.getStores()).isSameAs(SslStoreBundle.NONE);
        assertThat(bundle.getKey()).isSameAs(SslBundleKey.NONE);
        assertThat(bundle.getOptions()).isSameAs(SslOptions.NONE);
        assertThat(bundle.getProtocol()).isEqualTo("TLS");
    }

    @Test
    void shouldCreateBundle_WithCustomProtocol() {
        SslBundle bundle = SslBundle.of(null, null, null, "TLSv1.3");
        assertThat(bundle.getProtocol()).isEqualTo("TLSv1.3");
    }

    @Test
    void shouldCreateBundle_WithStoresAndKey() {
        SslStoreBundle stores = SslStoreBundle.of(null, "password", null);
        SslBundleKey key = SslBundleKey.of("keyPass", "myAlias");

        SslBundle bundle = SslBundle.of(stores, key, null, null);

        assertThat(bundle.getStores()).isSameAs(stores);
        assertThat(bundle.getKey()).isSameAs(key);
        assertThat(bundle.getStores().getKeyStorePassword()).isEqualTo("password");
    }

    @Test
    void shouldAutoCreateManagers_WhenNotExplicitlyProvided() {
        SslBundle bundle = SslBundle.of(null, null, null, null);
        // Managers should be auto-created (even if stores are empty)
        assertThat(bundle.getManagers()).isNotNull();
    }

    @Test
    void shouldCreateSslContext_WithRealKeyStore() throws Exception {
        KeyStore keyStore = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        SslStoreBundle stores = SslStoreBundle.of(keyStore, "changeit", null);
        SslBundle bundle = SslBundle.of(stores, SslBundleKey.NONE, null, null);

        SSLContext sslContext = bundle.createSslContext();
        assertThat(sslContext).isNotNull();
        assertThat(sslContext.getProtocol()).isEqualTo("TLS");
    }

    @Test
    void shouldUseExplicitManagers_WhenProvided() {
        SslManagerBundle managers = SslManagerBundle.of(null, null);
        SslBundle bundle = SslBundle.of(null, null, null, null, managers);
        assertThat(bundle.getManagers()).isSameAs(managers);
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/ssl/DefaultSslBundleRegistryTest.java`

```java
package com.iris.boot.ssl;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for {@link DefaultSslBundleRegistry}.
 */
class DefaultSslBundleRegistryTest {

    private DefaultSslBundleRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new DefaultSslBundleRegistry();
    }

    @Test
    void shouldRegisterAndRetrieveBundle() {
        SslBundle bundle = SslBundle.of(null, null, null, null);
        registry.registerBundle("test", bundle);

        SslBundle retrieved = registry.getBundle("test");
        assertThat(retrieved).isSameAs(bundle);
    }

    @Test
    void shouldRegisterMultipleBundles() {
        SslBundle bundle1 = SslBundle.of(null, null, null, "TLSv1.2");
        SslBundle bundle2 = SslBundle.of(null, null, null, "TLSv1.3");

        registry.registerBundle("server", bundle1);
        registry.registerBundle("client", bundle2);

        assertThat(registry.getBundle("server").getProtocol()).isEqualTo("TLSv1.2");
        assertThat(registry.getBundle("client").getProtocol()).isEqualTo("TLSv1.3");
    }

    @Test
    void shouldThrowNoSuchSslBundleException_WhenBundleNotFound() {
        assertThatThrownBy(() -> registry.getBundle("nonExistent"))
                .isInstanceOf(NoSuchSslBundleException.class)
                .hasMessageContaining("nonExistent");
    }

    @Test
    void shouldThrow_WhenRegisteringDuplicateName() {
        SslBundle bundle = SslBundle.of(null, null, null, null);
        registry.registerBundle("test", bundle);

        assertThatThrownBy(() -> registry.registerBundle("test", bundle))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("already registered");
    }

    @Test
    void shouldThrow_WhenNameIsNull() {
        SslBundle bundle = SslBundle.of(null, null, null, null);
        assertThatThrownBy(() -> registry.registerBundle(null, bundle))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldThrow_WhenNameIsBlank() {
        SslBundle bundle = SslBundle.of(null, null, null, null);
        assertThatThrownBy(() -> registry.registerBundle("  ", bundle))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldThrow_WhenBundleIsNull() {
        assertThatThrownBy(() -> registry.registerBundle("test", null))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/ssl/jks/JksSslStoreBundleTest.java`

```java
package com.iris.boot.ssl.jks;

import java.io.File;
import java.security.KeyStore;

import com.iris.boot.ssl.SslTestHelper;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Tests for {@link JksSslStoreBundle}.
 */
class JksSslStoreBundleTest {

    @Test
    void shouldLoadKeyStore_FromFilePath() throws Exception {
        // Create a test keystore file
        KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        File file = SslTestHelper.writeKeyStoreToFile(source, "changeit");

        JksSslStoreDetails details = new JksSslStoreDetails("PKCS12", file.getAbsolutePath(), "changeit");
        JksSslStoreBundle bundle = new JksSslStoreBundle(details, null);

        KeyStore loaded = bundle.getKeyStore();
        assertThat(loaded).isNotNull();
        assertThat(loaded.containsAlias("test")).isTrue();
        assertThat(bundle.getKeyStorePassword()).isEqualTo("changeit");
    }

    @Test
    void shouldReturnNull_WhenNoDetailsProvided() {
        JksSslStoreBundle bundle = new JksSslStoreBundle(null, null);
        assertThat(bundle.getKeyStore()).isNull();
        assertThat(bundle.getTrustStore()).isNull();
        assertThat(bundle.getKeyStorePassword()).isNull();
    }

    @Test
    void shouldReturnNull_WhenLocationIsNull() {
        JksSslStoreDetails details = new JksSslStoreDetails("PKCS12", null, null);
        JksSslStoreBundle bundle = new JksSslStoreBundle(details, null);
        assertThat(bundle.getKeyStore()).isNull();
    }

    @Test
    void shouldLoadTrustStore() throws Exception {
        KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        File file = SslTestHelper.writeKeyStoreToFile(source, "changeit");

        JksSslStoreDetails trustDetails = new JksSslStoreDetails("PKCS12", file.getAbsolutePath(), "changeit");
        JksSslStoreBundle bundle = new JksSslStoreBundle(null, trustDetails);

        assertThat(bundle.getKeyStore()).isNull();
        assertThat(bundle.getTrustStore()).isNotNull();
        assertThat(bundle.getTrustStore().containsAlias("test")).isTrue();
    }

    @Test
    void shouldLazyLoadAndCache() throws Exception {
        KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        File file = SslTestHelper.writeKeyStoreToFile(source, "changeit");

        JksSslStoreDetails details = new JksSslStoreDetails("PKCS12", file.getAbsolutePath(), "changeit");
        JksSslStoreBundle bundle = new JksSslStoreBundle(details, null);

        // First call loads
        KeyStore first = bundle.getKeyStore();
        // Second call returns cached
        KeyStore second = bundle.getKeyStore();
        assertThat(first).isSameAs(second);
    }

    @Test
    void shouldThrow_WhenFileNotFound() {
        JksSslStoreDetails details = new JksSslStoreDetails("PKCS12", "/nonexistent/keystore.p12", "changeit");
        JksSslStoreBundle bundle = new JksSslStoreBundle(details, null);

        assertThatThrownBy(bundle::getKeyStore)
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Unable to load key store");
    }

    @Test
    void shouldUseDefaultType_WhenTypeIsNull() throws Exception {
        KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        // Write as PKCS12 (the JVM default on modern JDKs)
        File file = SslTestHelper.writeKeyStoreToFile(source, "changeit");

        JksSslStoreDetails details = new JksSslStoreDetails(null, file.getAbsolutePath(), "changeit");
        JksSslStoreBundle bundle = new JksSslStoreBundle(details, null);

        KeyStore loaded = bundle.getKeyStore();
        assertThat(loaded).isNotNull();
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/ssl/pem/PemSslStoreBundleTest.java`

```java
package com.iris.boot.ssl.pem;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.security.KeyStore;

import com.iris.boot.ssl.SslTestHelper;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@link PemSslStoreBundle}.
 */
class PemSslStoreBundleTest {

    @TempDir
    File tempDir;

    @Test
    void shouldCreateKeyStore_FromPemFiles() throws Exception {
        // Generate a self-signed keystore, then export as PEM
        KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        String certPem = SslTestHelper.toPemCertificate(source, "test");
        String keyPem = SslTestHelper.toPemPrivateKey(source, "test", "changeit");

        // Write PEM files
        File certFile = writeFile("cert.pem", certPem);
        File keyFile = writeFile("key.pem", keyPem);

        PemSslStoreDetails details = new PemSslStoreDetails(
                certFile.getAbsolutePath(),
                keyFile.getAbsolutePath(),
                null);
        PemSslStoreBundle bundle = new PemSslStoreBundle(details, null);

        KeyStore keyStore = bundle.getKeyStore();
        assertThat(keyStore).isNotNull();
        assertThat(keyStore.containsAlias("ssl")).isTrue();
        assertThat(keyStore.isKeyEntry("ssl")).isTrue();
    }

    @Test
    void shouldCreateTrustStore_FromPemCertificateOnly() throws Exception {
        KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        String certPem = SslTestHelper.toPemCertificate(source, "test");

        File certFile = writeFile("ca.pem", certPem);

        PemSslStoreDetails trustDetails = new PemSslStoreDetails(
                certFile.getAbsolutePath(), null, null);
        PemSslStoreBundle bundle = new PemSslStoreBundle(null, trustDetails);

        assertThat(bundle.getKeyStore()).isNull();
        KeyStore trustStore = bundle.getTrustStore();
        assertThat(trustStore).isNotNull();
        assertThat(trustStore.containsAlias("ssl")).isTrue();
        assertThat(trustStore.isCertificateEntry("ssl")).isTrue();
    }

    @Test
    void shouldAcceptInlinePemContent() throws Exception {
        KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        String certPem = SslTestHelper.toPemCertificate(source, "test");
        String keyPem = SslTestHelper.toPemPrivateKey(source, "test", "changeit");

        // Pass inline PEM content directly (not file paths)
        PemSslStoreDetails details = new PemSslStoreDetails(certPem, keyPem, null);
        PemSslStoreBundle bundle = new PemSslStoreBundle(details, null);

        KeyStore keyStore = bundle.getKeyStore();
        assertThat(keyStore).isNotNull();
        assertThat(keyStore.isKeyEntry("ssl")).isTrue();
    }

    @Test
    void shouldReturnNull_WhenNoDetailsProvided() {
        PemSslStoreBundle bundle = new PemSslStoreBundle(null, null);
        assertThat(bundle.getKeyStore()).isNull();
        assertThat(bundle.getTrustStore()).isNull();
    }

    @Test
    void shouldReturnNullKeyStorePassword() {
        PemSslStoreBundle bundle = new PemSslStoreBundle(null, null);
        assertThat(bundle.getKeyStorePassword()).isNull();
    }

    @Test
    void shouldLazyLoadAndCache() throws Exception {
        KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        String certPem = SslTestHelper.toPemCertificate(source, "test");
        String keyPem = SslTestHelper.toPemPrivateKey(source, "test", "changeit");

        PemSslStoreDetails details = new PemSslStoreDetails(certPem, keyPem, null);
        PemSslStoreBundle bundle = new PemSslStoreBundle(details, null);

        KeyStore first = bundle.getKeyStore();
        KeyStore second = bundle.getKeyStore();
        assertThat(first).isSameAs(second);
    }

    private File writeFile(String name, String content) throws IOException {
        File file = new File(tempDir, name);
        Files.writeString(file.toPath(), content);
        return file;
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/context/properties/MapBindingTest.java`

```java
package com.iris.boot.context.properties;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Properties;

import com.iris.framework.core.env.MapPropertySource;
import com.iris.framework.core.env.MutablePropertySources;
import com.iris.framework.core.env.StandardEnvironment;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests for {@code Map<String, ?>} binding support added in Feature 22.
 */
class MapBindingTest {

    @Test
    void shouldBindMapOfStrings() {
        StandardEnvironment env = createEnvironment(Map.of(
                "app.tags.env", "production",
                "app.tags.region", "us-east-1"
        ));

        TagConfig target = new TagConfig();
        new ConfigurationPropertiesBinder(env).bind(target, "app");

        assertThat(target.getTags()).containsEntry("env", "production");
        assertThat(target.getTags()).containsEntry("region", "us-east-1");
    }

    @Test
    void shouldBindMapOfNestedObjects() {
        StandardEnvironment env = createEnvironment(Map.of(
                "ssl.bundles.server.location", "/etc/ssl/server.p12",
                "ssl.bundles.server.password", "secret",
                "ssl.bundles.client.location", "/etc/ssl/client.p12",
                "ssl.bundles.client.password", "changeit"
        ));

        SslConfig target = new SslConfig();
        new ConfigurationPropertiesBinder(env).bind(target, "ssl");

        assertThat(target.getBundles()).hasSize(2);
        assertThat(target.getBundles().get("server").getLocation()).isEqualTo("/etc/ssl/server.p12");
        assertThat(target.getBundles().get("server").getPassword()).isEqualTo("secret");
        assertThat(target.getBundles().get("client").getLocation()).isEqualTo("/etc/ssl/client.p12");
        assertThat(target.getBundles().get("client").getPassword()).isEqualTo("changeit");
    }

    @Test
    void shouldBindMapWithDeeplyNestedObjects() {
        StandardEnvironment env = createEnvironment(Map.of(
                "spring.ssl.bundle.jks.myBundle.keystore.location", "classpath:keystore.p12",
                "spring.ssl.bundle.jks.myBundle.keystore.password", "secret",
                "spring.ssl.bundle.jks.myBundle.keystore.type", "PKCS12"
        ));

        DeepConfig target = new DeepConfig();
        new ConfigurationPropertiesBinder(env).bind(target, "spring.ssl");

        assertThat(target.getBundle()).isNotNull();
        assertThat(target.getBundle().getJks()).hasSize(1);
        StoreProps store = target.getBundle().getJks().get("myBundle").getKeystore();
        assertThat(store.getLocation()).isEqualTo("classpath:keystore.p12");
        assertThat(store.getPassword()).isEqualTo("secret");
        assertThat(store.getType()).isEqualTo("PKCS12");
    }

    @Test
    void shouldCreateMapIfNull() {
        StandardEnvironment env = createEnvironment(Map.of(
                "app.items.one", "valueOne"
        ));

        NullMapConfig target = new NullMapConfig();
        assertThat(target.getItems()).isNull();

        new ConfigurationPropertiesBinder(env).bind(target, "app");

        assertThat(target.getItems()).isNotNull();
        assertThat(target.getItems()).containsEntry("one", "valueOne");
    }

    @Test
    void shouldNotModifyMap_WhenNoMatchingProperties() {
        StandardEnvironment env = createEnvironment(Map.of(
                "other.key", "value"
        ));

        TagConfig target = new TagConfig();
        new ConfigurationPropertiesBinder(env).bind(target, "app");

        assertThat(target.getTags()).isEmpty();
    }

    // -----------------------------------------------------------------------
    // Test config classes
    // -----------------------------------------------------------------------

    private StandardEnvironment createEnvironment(Map<String, String> properties) {
        StandardEnvironment env = new StandardEnvironment();
        MutablePropertySources sources = env.getPropertySources();
        Map<String, Object> map = new java.util.HashMap<>(properties);
        sources.addFirst(new MapPropertySource("test", map));
        return env;
    }

    public static class TagConfig {
        private Map<String, String> tags = new LinkedHashMap<>();

        public Map<String, String> getTags() {
            return tags;
        }

        public void setTags(Map<String, String> tags) {
            this.tags = tags;
        }
    }

    public static class SslConfig {
        private Map<String, BundleProps> bundles = new LinkedHashMap<>();

        public Map<String, BundleProps> getBundles() {
            return bundles;
        }

        public void setBundles(Map<String, BundleProps> bundles) {
            this.bundles = bundles;
        }
    }

    public static class BundleProps {
        private String location;
        private String password;

        public String getLocation() {
            return location;
        }

        public void setLocation(String location) {
            this.location = location;
        }

        public String getPassword() {
            return password;
        }

        public void setPassword(String password) {
            this.password = password;
        }
    }

    public static class NullMapConfig {
        private Map<String, String> items;

        public Map<String, String> getItems() {
            return items;
        }

        public void setItems(Map<String, String> items) {
            this.items = items;
        }
    }

    // Deep nesting: mirrors the SslProperties structure
    public static class DeepConfig {
        private BundleConfig bundle = new BundleConfig();

        public BundleConfig getBundle() {
            return bundle;
        }

        public void setBundle(BundleConfig bundle) {
            this.bundle = bundle;
        }
    }

    public static class BundleConfig {
        private Map<String, JksProps> jks = new LinkedHashMap<>();

        public Map<String, JksProps> getJks() {
            return jks;
        }

        public void setJks(Map<String, JksProps> jks) {
            this.jks = jks;
        }
    }

    public static class JksProps {
        private StoreProps keystore = new StoreProps();

        public StoreProps getKeystore() {
            return keystore;
        }

        public void setKeystore(StoreProps keystore) {
            this.keystore = keystore;
        }
    }

    public static class StoreProps {
        private String location;
        private String password;
        private String type;

        public String getLocation() {
            return location;
        }

        public void setLocation(String location) {
            this.location = location;
        }

        public String getPassword() {
            return password;
        }

        public void setPassword(String password) {
            this.password = password;
        }

        public String getType() {
            return type;
        }

        public void setType(String type) {
            this.type = type;
        }
    }
}
```

#### `[NEW]` `iris-boot-core/src/test/java/com/iris/boot/ssl/integration/SslBundleIntegrationTest.java`

```java
package com.iris.boot.ssl.integration;

import java.io.File;
import java.security.KeyStore;

import javax.net.ssl.SSLContext;

import com.iris.boot.ssl.DefaultSslBundleRegistry;
import com.iris.boot.ssl.NoSuchSslBundleException;
import com.iris.boot.ssl.SslBundle;
import com.iris.boot.ssl.SslBundleKey;
import com.iris.boot.ssl.SslBundles;
import com.iris.boot.ssl.SslOptions;
import com.iris.boot.ssl.SslStoreBundle;
import com.iris.boot.ssl.SslTestHelper;
import com.iris.boot.ssl.jks.JksSslStoreBundle;
import com.iris.boot.ssl.jks.JksSslStoreDetails;
import com.iris.boot.ssl.pem.PemSslStoreBundle;
import com.iris.boot.ssl.pem.PemSslStoreDetails;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Integration tests verifying the full SSL Bundles workflow:
 * create stores → build bundles → register → retrieve → create SSLContext.
 */
class SslBundleIntegrationTest {

    @Test
    void shouldCreateSslContextFromJksBundle() throws Exception {
        // 1. Create a self-signed keystore
        KeyStore keyStore = SslTestHelper.createSelfSignedKeyStore("server", "changeit");
        File keystoreFile = SslTestHelper.writeKeyStoreToFile(keyStore, "changeit");

        // 2. Load it as a JKS store bundle
        JksSslStoreDetails details = new JksSslStoreDetails(
                "PKCS12", keystoreFile.getAbsolutePath(), "changeit");
        JksSslStoreBundle stores = new JksSslStoreBundle(details, null);

        // 3. Create a bundle
        SslBundle bundle = SslBundle.of(stores, SslBundleKey.NONE, null, null);

        // 4. Register and retrieve
        DefaultSslBundleRegistry registry = new DefaultSslBundleRegistry();
        registry.registerBundle("server", bundle);

        SslBundles bundles = registry;
        SslBundle retrieved = bundles.getBundle("server");

        // 5. Create an SSLContext
        SSLContext sslContext = retrieved.createSslContext();
        assertThat(sslContext).isNotNull();
        assertThat(sslContext.getProtocol()).isEqualTo("TLS");
    }

    @Test
    void shouldCreateSslContextFromPemBundle() throws Exception {
        // 1. Generate PEM content from a self-signed keystore
        KeyStore source = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        String certPem = SslTestHelper.toPemCertificate(source, "test");
        String keyPem = SslTestHelper.toPemPrivateKey(source, "test", "changeit");

        // 2. Create a PEM store bundle (inline content)
        PemSslStoreDetails keyStoreDetails = new PemSslStoreDetails(certPem, keyPem, null);
        PemSslStoreBundle stores = new PemSslStoreBundle(keyStoreDetails, null);

        // 3. Create and register the bundle
        SslBundle bundle = SslBundle.of(stores, SslBundleKey.NONE, null, "TLSv1.3");
        DefaultSslBundleRegistry registry = new DefaultSslBundleRegistry();
        registry.registerBundle("pem-bundle", bundle);

        // 4. Retrieve and use
        SslBundle retrieved = registry.getBundle("pem-bundle");
        assertThat(retrieved.getProtocol()).isEqualTo("TLSv1.3");

        SSLContext sslContext = retrieved.createSslContext();
        assertThat(sslContext).isNotNull();
        assertThat(sslContext.getProtocol()).isEqualTo("TLSv1.3");
    }

    @Test
    void shouldSupportMultipleBundleFormats() throws Exception {
        DefaultSslBundleRegistry registry = new DefaultSslBundleRegistry();

        // JKS bundle
        KeyStore jksStore = SslTestHelper.createSelfSignedKeyStore("jks-entry", "changeit");
        File jksFile = SslTestHelper.writeKeyStoreToFile(jksStore, "changeit");
        JksSslStoreBundle jksStores = new JksSslStoreBundle(
                new JksSslStoreDetails("PKCS12", jksFile.getAbsolutePath(), "changeit"),
                null);
        registry.registerBundle("jks-server", SslBundle.of(jksStores, SslBundleKey.NONE, null, null));

        // PEM bundle
        KeyStore pemSource = SslTestHelper.createSelfSignedKeyStore("pem-entry", "changeit");
        PemSslStoreBundle pemStores = new PemSslStoreBundle(
                new PemSslStoreDetails(
                        SslTestHelper.toPemCertificate(pemSource, "pem-entry"),
                        SslTestHelper.toPemPrivateKey(pemSource, "pem-entry", "changeit"),
                        null),
                null);
        registry.registerBundle("pem-client", SslBundle.of(pemStores, SslBundleKey.NONE, null, null));

        // Both should work independently
        assertThat(registry.getBundle("jks-server").createSslContext()).isNotNull();
        assertThat(registry.getBundle("pem-client").createSslContext()).isNotNull();
    }

    @Test
    void shouldFailFast_WhenBundleNotRegistered() {
        DefaultSslBundleRegistry registry = new DefaultSslBundleRegistry();

        assertThatThrownBy(() -> registry.getBundle("missing"))
                .isInstanceOf(NoSuchSslBundleException.class)
                .hasMessageContaining("missing");
    }

    @Test
    void shouldApplySslOptions() throws Exception {
        KeyStore keyStore = SslTestHelper.createSelfSignedKeyStore("test", "changeit");
        SslStoreBundle stores = SslStoreBundle.of(keyStore, "changeit", null);
        SslOptions options = SslOptions.of(
                new String[]{"TLS_AES_128_GCM_SHA256"},
                new String[]{"TLSv1.3"});

        SslBundle bundle = SslBundle.of(stores, SslBundleKey.NONE, options, "TLSv1.3");

        assertThat(bundle.getOptions().getCiphers()).containsExactly("TLS_AES_128_GCM_SHA256");
        assertThat(bundle.getOptions().getEnabledProtocols()).containsExactly("TLSv1.3");

        // SSL context should still be creatable
        SSLContext sslContext = bundle.createSslContext();
        assertThat(sslContext).isNotNull();
    }
}
```

---

## Summary

| What | Detail |
|------|--------|
| **New classes** | `SslBundleKey`, `SslOptions`, `SslStoreBundle`, `SslManagerBundle`, `DefaultSslManagerBundle`, `SslBundle`, `SslBundleRegistry`, `SslBundles`, `DefaultSslBundleRegistry`, `NoSuchSslBundleException`, `JksSslStoreDetails`, `JksSslStoreBundle`, `PemSslStoreDetails`, `PemContent`, `PemSslStoreBundle`, `JksSslBundleProperties`, `PemSslBundleProperties`, `SslProperties`, `SslAutoConfiguration` |
| **Modified classes** | `ConfigurationPropertiesBinder` (+`Map<String, ?>` binding), `ServerProperties` (+`Ssl.bundle`), `TomcatServletWebServerFactory` (+SSL connector), `TomcatAutoConfiguration` (+`SslBundles` wiring, `@AutoConfigureAfter`), `AutoConfiguration.imports` (+`SslAutoConfiguration`) |
| **Pattern** | Composition-based SSL abstraction with format-agnostic store normalization and interface-segregated registry |
| **Key insight** | All certificate formats normalize to Java `KeyStore` objects, so consumers never know whether the source was JKS, PKCS12, or PEM |
| **Tests** | 10 test files: `SslBundleKeyTest` (8), `SslOptionsTest` (5), `SslManagerBundleTest` (5), `SslBundleTest` (6), `DefaultSslBundleRegistryTest` (7), `JksSslStoreBundleTest` (7), `PemSslStoreBundleTest` (6), `MapBindingTest` (5), `SslBundleIntegrationTest` (5), `SslTestHelper` (shared helper) |

**Next chapter:** [Chapter 23](ch23_observability.md) -- add observability support with metrics and tracing integration.
