# Project: Secure-Secrets Library + Sample App

This project demonstrates two major versions of a Secure Secrets & Configuration Access Library and a single sample application that first uses v1.0.0 (Java 8) and then shows how to migrate it to v2.0.0 (Java 21).

```
secure-secrets-project/
├── secure-secrets-v1/         # Java 8 library (v1.0.0)
│   ├── pom.xml
│   └── src/main/java/com/example/secrets/v1/
│       ├── SecretProvider.java
│       ├── VaultSecretProvider.java
│       ├── AwsSecretsManagerProvider.java
│       ├── CachedSecret.java
│       ├── SecretCache.java
│       ├── LoggerRedactor.java
│       └── SecretsManager.java
├── secure-secrets-v2/         # Java 21 library (v2.0.0, breaking changes)
│   ├── pom.xml
│   └── src/main/java/com/example/secrets/v2/
│       ├── SecretClient.java
│       ├── ProviderStrategy.java
│       ├── VaultProvider.java
│       ├── AwsProvider.java
│       ├── CachePolicy.java
│       ├── TimeBasedPolicy.java
│       └── SecretsException.java
└── sample-app/                # Demonstrates v1 → v2 migration
    ├── pom.xml
    └── src/main/java/com/example/app/
        ├── AppV1.java
        └── AppV2.java
```

---

## Library v1.0.0 (Java 8)

### pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 \
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>secure-secrets</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
  </properties>
</project>
```

### SecretProvider.java
```java
package com.example.secrets.v1;

public interface SecretProvider {
    String getSecret(String key) throws Exception;
}
```

### VaultSecretProvider.java
```java
package com.example.secrets.v1;

public class VaultSecretProvider implements SecretProvider {
    @Override
    public String getSecret(String key) throws Exception {
        // Placeholder: call Vault HTTP API
        return "vault-secret-for-" + key;
    }
}
```

### AwsSecretsManagerProvider.java
```java
package com.example.secrets.v1;

public class AwsSecretsManagerProvider implements SecretProvider {
    @Override
    public String getSecret(String key) throws Exception {
        // Placeholder: call AWS Secrets Manager SDK v1
        return "aws-secret-for-" + key;
    }
}
```

### CachedSecret.java
```java
package com.example.secrets.v1;

import java.util.Date;

public class CachedSecret {
    private final String value;
    private final Date expiry;

    public CachedSecret(String value, Date expiry) {
        this.value = value;
        this.expiry = expiry;
    }

    public String getValue() { return value; }
    public Date getExpiry() { return expiry; }
    public boolean isExpired() {
        return expiry.before(new Date());
    }
}
```

### SecretCache.java
```java
package com.example.secrets.v1;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

public class SecretCache {
    private final Map<String, CachedSecret> cache =
        Collections.synchronizedMap(new HashMap<>());

    public void put(String key, String value, long ttlMillis) {
        cache.put(key, new CachedSecret(value,
            new java.util.Date(System.currentTimeMillis() + ttlMillis)));
    }

    public CachedSecret get(String key) {
        return cache.get(key);
    }

    public Set<String> keys() {
        return cache.keySet();
    }
}
```

### LoggerRedactor.java
```java
package com.example.secrets.v1;

public class LoggerRedactor {
    public static String redact(String message) {
        return message.replaceAll("(?i)(secret\\s*:\\s*)(\\S+)", "$1******");
    }
}
```

### SecretsManager.java
```java
package com.example.secrets.v1;

import java.util.Timer;
import java.util.TimerTask;

public class SecretsManager {
    private final SecretProvider provider;
    private final SecretCache cache = new SecretCache();
    private final long ttlMillis;
    private final Timer refreshTimer = new Timer(true);

    public SecretsManager(SecretProvider provider, long ttlMillis) {
        this.provider = provider;
        this.ttlMillis = ttlMillis;
        startRefreshTask();
    }

    public String getSecret(String key) {
        CachedSecret cs = cache.get(key);
        if (cs == null || cs.isExpired()) {
            try {
                String secret = provider.getSecret(key);
                cache.put(key, secret, ttlMillis);
                return secret;
            } catch (Exception e) {
                if (cs != null) return cs.getValue();
                throw new RuntimeException("Failed to fetch secret", e);
            }
        }
        return cs.getValue();
    }

    @Deprecated
    public void stop() {
        refreshTimer.cancel();
        refreshTimer.purge();
    }

    private void startRefreshTask() {
        refreshTimer.scheduleAtFixedRate(new TimerTask() {
            @Override public void run() {
                for (String key : cache.keys()) {
                    try {
                        String secret = provider.getSecret(key);
                        cache.put(key, secret, ttlMillis);
                        System.out.println("Refreshed secret: " + key);
                    } catch (Exception e) {
                        System.err.println("Refresh failed: " + key);
                    }
                }
            }
        }, ttlMillis, ttlMillis);
    }
}
```

---

## Library v2.0.0 (Java 21, Breaking Changes)

### pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 \
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>secret-client</artifactId>
  <version>2.0.0</version>
  <packaging>jar</packaging>

  <properties>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
  </properties>
</project>
```

### SecretClient.java
```java
package com.example.secrets.v2;

import java.util.concurrent.CompletionStage;

public class SecretClient {
    private final ProviderStrategy provider;
    private final CachePolicy cache;

    private SecretClient(ProviderStrategy provider, CachePolicy cache) {
        this.provider = provider;
        this.cache = cache;
    }

    public static Builder builder() {
        return new Builder();
    }

    public CompletionStage<String> fetchSecretAsync(String key) {
        return provider.retrieve(key)
            .thenApply(secret -> {
                cache.put(key, secret);
                return secret;
            });
    }

    public void close() {
        cache.shutdown();
    }

    public static class Builder {
        private ProviderStrategy provider;
        private CachePolicy cache;

        public Builder withProvider(ProviderStrategy p) {
            this.provider = p;
            return this;
        }

        public Builder withCachePolicy(CachePolicy c) {
            this.cache = c;
            return this;
        }

        public SecretClient build() {
            if (provider == null || cache == null) {
                throw new IllegalStateException("Provider and CachePolicy required");
            }
            return new SecretClient(provider, cache);
        }
    }
}
```

### ProviderStrategy.java
```java
package com.example.secrets.v2;

import java.util.concurrent.CompletionStage;

public sealed interface ProviderStrategy
    permits VaultProvider, AwsProvider {
    CompletionStage<String> retrieve(String key);
}
```

### VaultProvider.java
```java
package com.example.secrets.v2;

import java.util.concurrent.CompletableFuture;

public final class VaultProvider implements ProviderStrategy {
    @Override
    public CompletionStage<String> retrieve(String key) {
        return CompletableFuture.supplyAsync(() -> "vault-secret-for-" + key);
    }
}
```

### AwsProvider.java
```java
package com.example.secrets.v2;

import java.util.concurrent.CompletableFuture;

public final class AwsProvider implements ProviderStrategy {
    @Override
    public CompletionStage<String> retrieve(String key) {
        return CompletableFuture.supplyAsync(() -> "aws-secret-for-" + key);
    }
}
```

### CachePolicy.java
```java
package com.example.secrets.v2;

import java.util.concurrent.CompletionStage;

public sealed interface CachePolicy
    permits TimeBasedPolicy {
    void put(String key, String value);
    CompletionStage<String> get(String key);
    void shutdown();
}
```

### TimeBasedPolicy.java
```java
package com.example.secrets.v2;

import java.time.Duration;
import java.util.concurrent.*;

public final class TimeBasedPolicy implements CachePolicy {
    private final ConcurrentHashMap<String, String> store = new ConcurrentHashMap<>();
    private final ScheduledExecutorService janitor =
        Executors.newSingleThreadScheduledExecutor(r -> Thread.ofVirtual().factory().newThread(r));

    public TimeBasedPolicy(Duration ttl) {
        janitor.scheduleAtFixedRate(this::evictExpired,
            ttl.toSeconds(), ttl.toSeconds(), TimeUnit.SECONDS);
    }

    @Override
    public void put(String key, String value) {
        store.put(key, value);
    }

    @Override
    public CompletionStage<String> get(String key) {
        return CompletableFuture.completedFuture(store.get(key));
    }

    private void evictExpired() {
        // Eviction logic here
    }

    @Override
    public void shutdown() {
        janitor.shutdown();
    }
}
```

### SecretsException.java
```java
package com.example.secrets.v2;

public class SecretsException extends RuntimeException {
    public SecretsException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

## Sample App: v1 → v2 Migration

### pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 \
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>sample-app</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <properties>
    <!-- Start with Java 8, then switch to 21 -->
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
  </properties>

  <dependencies>
    <!-- Initially v1 library -->
    <dependency>
      <groupId>com.example</groupId>
      <artifactId>secure-secrets</artifactId>
      <version>1.0.0</version>
    </dependency>
  </dependencies>
</project>
```

### AppV1.java
```java
package com.example.app;

import com.example.secrets.v1.*;

public class AppV1 {
    public static void main(String[] args) {
        SecretProvider provider = new VaultSecretProvider();
        SecretsManager mgr = new SecretsManager(provider, 5000);

        String pwd = mgr.getSecret("db.password");
        System.out.println(LoggerRedactor.redact("db.password: " + pwd));

        mgr.stop();
    }
}
```

### Migration Steps
1. Update POM to Java 21 and replace dependency with v2:
   ```xml
   <maven.compiler.source>21</maven.compiler.source>
   <maven.compiler.target>21</maven.compiler.target>
   …
   <artifactId>secret-client</artifactId>
   <version>2.0.0</version>
   ```
2. Refactor imports: `com.example.secrets.v1` → `com.example.secrets.v2`
3. Replace `SecretsManager` usage with builder API
4. Handle `CompletionStage` instead of direct return
5. Call `close()` instead of `stop()`

### AppV2.java
```java
package com.example.app;

import com.example.secrets.v2.*;
import java.time.Duration;

public class AppV2 {
    public static void main(String[] args) {
        SecretClient client = SecretClient.builder()
            .withProvider(new VaultProvider())
            .withCachePolicy(new TimeBasedPolicy(Duration.ofSeconds(5)))
            .build();

        client.fetchSecretAsync("db.password")
              .thenAccept(pwd -> System.out.println("db.password: " + pwd))
              .exceptionally(ex -> { ex.printStackTrace(); return null; });

        client.close();
    }
}
```
