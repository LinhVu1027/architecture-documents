# Chapter 7: Building the ZaloBot Properties Class

> Complete, compilable `ZaloBotProperties.java` — mapping every property to its SDK counterpart.

---

## The Complete Class

```java
package dev.linhvu.zalobot.autoconfigure;

import java.time.Duration;

import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * Configuration properties for ZaloBot.
 *
 * @see dev.linhvu.zalobot.client.ZaloBotClient
 * @see dev.linhvu.zalobot.listener.ContainerProperties
 */
@ConfigurationProperties("zalobot")
public class ZaloBotProperties {

    /**
     * Bot token for authenticating with the Zalo Bot API.
     * Required for auto-configuration to activate.
     */
    private String botToken;

    private final Client client = new Client();

    private final Listener listener = new Listener();

    public String getBotToken() {
        return this.botToken;
    }

    public void setBotToken(String botToken) {
        this.botToken = botToken;
    }

    public Client getClient() {
        return this.client;
    }

    public Listener getListener() {
        return this.listener;
    }

    /**
     * Client connection properties.
     * Maps to {@link dev.linhvu.zalobot.client.ZaloBotUrl}.
     */
    public static class Client {

        /**
         * URL scheme for the Zalo Bot API.
         */
        private String scheme = "https";

        /**
         * Hostname of the Zalo Bot API.
         */
        private String host = "bot-api.zaloplatforms.com";

        /**
         * Port of the Zalo Bot API.
         */
        private int port = 443;

        public String getScheme() {
            return this.scheme;
        }

        public void setScheme(String scheme) {
            this.scheme = scheme;
        }

        public String getHost() {
            return this.host;
        }

        public void setHost(String host) {
            this.host = host;
        }

        public int getPort() {
            return this.port;
        }

        public void setPort(int port) {
            this.port = port;
        }
    }

    /**
     * Listener container properties.
     * Maps to {@link dev.linhvu.zalobot.listener.ContainerProperties}.
     */
    public static class Listener {

        /**
         * Whether the listener container is enabled.
         * When false, no listener container is created even if an UpdateListener bean exists.
         */
        private boolean enabled = true;

        /**
         * Timeout for long-polling the Zalo API for updates.
         * Maps to ContainerProperties.pollTimeout.
         */
        private Duration pollTimeout = Duration.ofSeconds(30);

        /**
         * Interval between poll cycles.
         * Maps to ContainerProperties.pollInterval.
         */
        private Duration pollInterval = Duration.ofSeconds(0);

        /**
         * Maximum time to wait for the listener container to shut down gracefully.
         * Maps to ContainerProperties.shutdownTimeout.
         */
        private Duration shutdownTimeout = Duration.ofSeconds(10);

        /**
         * Initial back-off interval after a poll error.
         * Maps to ContainerProperties.backOffInterval.
         */
        private Duration backOffInterval = Duration.ofSeconds(1);

        /**
         * Maximum back-off interval after repeated poll errors.
         * Maps to ContainerProperties.maxBackOffInterval.
         */
        private Duration maxBackOffInterval = Duration.ofSeconds(30);

        /**
         * Number of concurrent listener containers.
         * 1 creates a single ZaloUpdateListenerContainer.
         * >1 creates a ConcurrentUpdateListenerContainer with this many children.
         */
        private int concurrency = 1;

        public boolean isEnabled() {
            return this.enabled;
        }

        public void setEnabled(boolean enabled) {
            this.enabled = enabled;
        }

        public Duration getPollTimeout() {
            return this.pollTimeout;
        }

        public void setPollTimeout(Duration pollTimeout) {
            this.pollTimeout = pollTimeout;
        }

        public Duration getPollInterval() {
            return this.pollInterval;
        }

        public void setPollInterval(Duration pollInterval) {
            this.pollInterval = pollInterval;
        }

        public Duration getShutdownTimeout() {
            return this.shutdownTimeout;
        }

        public void setShutdownTimeout(Duration shutdownTimeout) {
            this.shutdownTimeout = shutdownTimeout;
        }

        public Duration getBackOffInterval() {
            return this.backOffInterval;
        }

        public void setBackOffInterval(Duration backOffInterval) {
            this.backOffInterval = backOffInterval;
        }

        public Duration getMaxBackOffInterval() {
            return this.maxBackOffInterval;
        }

        public void setMaxBackOffInterval(Duration maxBackOffInterval) {
            this.maxBackOffInterval = maxBackOffInterval;
        }

        public int getConcurrency() {
            return this.concurrency;
        }

        public void setConcurrency(int concurrency) {
            this.concurrency = concurrency;
        }
    }
}
```

---

## Property-to-SDK Mapping Table

| Property | Type | Default | Maps To |
|---|---|---|---|
| `zalobot.bot-token` | `String` | *(required)* | `ZaloBotClient.Builder.botToken()` |
| `zalobot.client.scheme` | `String` | `"https"` | `ZaloBotUrl.scheme` (`ZaloBotUrl.java:4`) |
| `zalobot.client.host` | `String` | `"bot-api.zaloplatforms.com"` | `ZaloBotUrl.host` (`ZaloBotUrl.java:5`) |
| `zalobot.client.port` | `int` | `443` | `ZaloBotUrl.port` (`ZaloBotUrl.java:6`) |
| `zalobot.listener.enabled` | `boolean` | `true` | Controls listener container creation |
| `zalobot.listener.poll-timeout` | `Duration` | `30s` | `ContainerProperties.pollTimeout` (`ContainerProperties.java:8`) |
| `zalobot.listener.poll-interval` | `Duration` | `0s` | `ContainerProperties.pollInterval` (`ContainerProperties.java:9`) |
| `zalobot.listener.shutdown-timeout` | `Duration` | `10s` | `ContainerProperties.shutdownTimeout` (`ContainerProperties.java:10`) |
| `zalobot.listener.back-off-interval` | `Duration` | `1s` | `ContainerProperties.backOffInterval` (`ContainerProperties.java:11`) |
| `zalobot.listener.max-back-off-interval` | `Duration` | `30s` | `ContainerProperties.maxBackOffInterval` (`ContainerProperties.java:12`) |
| `zalobot.listener.concurrency` | `int` | `1` | `1` = single container, `>1` = concurrent container |

---

## Complete `application.yml` with All Options

```yaml
zalobot:
  # Required — auto-configuration only activates when this is set
  bot-token: ${ZALO_BOT_TOKEN}

  # Client connection settings (all optional — sensible defaults)
  client:
    scheme: https                        # default: "https"
    host: bot-api.zaloplatforms.com      # default: "bot-api.zaloplatforms.com"
    port: 443                            # default: 443

  # Listener container settings (all optional)
  listener:
    enabled: true                        # default: true — set false to disable
    poll-timeout: 30s                    # default: 30s — long-poll timeout
    poll-interval: 0s                    # default: 0s — delay between polls
    shutdown-timeout: 10s                # default: 10s — graceful shutdown wait
    back-off-interval: 1s               # default: 1s — initial error backoff
    max-back-off-interval: 30s           # default: 30s — max error backoff
    concurrency: 1                       # default: 1 — concurrent containers
```

---

## Design Decisions

### Why Nested Static Classes (Not Separate Files)?

Following the Spring Boot convention (see `KafkaProperties.java` which has nested `Consumer`, `Producer`, `Admin`, `Listener`, etc.):
- Single file for all properties related to one module
- YAML hierarchy maps directly to class hierarchy
- IDE navigation: `ZaloBotProperties.Client` vs a separate `ZaloBotClientProperties` class

### Why Mutable Setters (Not Records)?

`@ConfigurationProperties` with setter binding (the default) requires:
- A no-arg constructor
- Setter methods for each property

Records are supported via `@ConstructorBinding`, but mutable setters are the established convention for Spring Boot properties classes and offer better IDE support for autocomplete.

### Why Duplicate Defaults?

The defaults in `ZaloBotProperties` mirror the SDK defaults in `ZaloBotUrl.java:8` and `ContainerProperties.java:8-12`. This means:
- Users see correct defaults in IDE autocomplete
- If a user doesn't set a property, they get the same behavior as using the SDK directly
- The properties class is the single source of truth for auto-configured defaults

---

## What's Next?

With the properties class ready, Chapter 8 builds the auto-configuration classes that consume these properties and create beans.

```
Chapter 7  ←── YOU ARE HERE: "Build it: properties"
Chapter 8  ──► "Build it: auto-config classes"
```
