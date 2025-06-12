# Enterprise Audit Logging & Compliance Trail Library

## Background & Motivation
In large enterprise environments, consistent and traceable audit logging is critical for:

- **Compliance**: Meeting regulatory requirements such as SOX, GDPR, and HIPAA mandates detailed records of who did what, when, and why.
- **Security & Forensics**: In the event of incidents, audit trails help reconstruct actions and detect unauthorized access or data manipulation.
- **Operational Visibility**: Understanding system behavior, configuration changes, and user actions across microservices.

Out-of-the-box logging frameworks vary from team to team, leading to inconsistent formats, missing metadata, and manual efforts to centralize audit data. This library offers a **standardized, reusable solution** to emit structured audit events, integrate seamlessly with SLF4J contexts, and support both synchronous and asynchronous backends.

The project evolves in two versions:

1. **v1 (Java 8)**: A baseline implementation using POJOs, SLF4J MDC, synchronous console or pluggable backends, and simple redaction filters.
2. **v2 (Java 21)**: Modernized with `record` types, `ScopedValue` for context propagation, virtual-thread based async logging, annotation-driven redaction, and an SPI-friendly architecture for future telemetry integration.

---

## Repository Structure
```
enterprise-audit-ecosystem/

├── enterprise-audit-logger/
│   ├── v1/         ← Java 8 library (1.0.0)
│   └── v2/         ← Java 21 library (2.0.0)
├── audit-app-demo/ ← Consumer application showcasing usage of v1 and v2
├── README.md       ← This overview and setup instructions
└── LICENSE
```

---

## Version 1 (Java 8)

### Goals & Features
- **Uniform AuditEvent** model via POJO
- **SLF4J MDC** for propagating `user` and `service`
- **SimpleAuditLogger** for sync console output
- **Redactor** filters sensitive metadata keys (e.g. `password`)

### v1 Code Snippets

#### pom.xml
```xml
<!-- ... Java 8 compiler settings, version 1.0.0 ... -->
```  

#### AuditEvent.java (POJO)
```java
// fields, getters, toString(), timestamp in constructor
```

#### SimpleAuditLogger.java
```java
System.out.println("[AUDIT] " + event);
```

#### Redactor.java
```java
metadata.replaceAll((k,v) -> k.contains("password") ? "[REDACTED]" : v);
```

---

## Version 2 (Java 21 Modern)

### Improvements & Enhancements
- **`record`** for concise immutable data model
- **`ScopedValue`** for type-safe context propagation, replacing MDC
- **AsyncConsoleAuditLogger** using **virtual threads** for non-blocking logging
- **`@Redact`** annotation-driven redaction via reflection
- **SPI-ready** design for future Kafka/OTEL/HTTP backends

### v2 Code Snippets

#### pom.xml
```xml
<!-- ... Java 21 compiler settings, version 2.0.0 ... -->
```

#### AuditEvent.java (record)
```java
public record AuditEvent(..., Instant timestamp, Map<String,Object> metadata) {}
```

#### AsyncConsoleAuditLogger.java
```java
executor = Executors.newVirtualThreadPerTaskExecutor();
CompletableFuture.runAsync(() -> log(event), executor);
```

#### Redact.java + Redactor.java
```java
// @Redact on metadata fields, Redactor uses reflection to redact
```

---

## Demo Application: Migration v1 → v2

The `audit-app-demo` folder contains two entry points:

1. **MainAppV1.java** (depends on v1):
   - Uses `MDC.put(...)`
   - Builds a `Map<String,Object>` of metadata
   - Calls `new SimpleAuditLogger().log(...)`

2. **MainAppV2.java** (depends on v2):
   - Uses `ScopedValue.where(...)` for context
   - Defines a `Metadata` class with `@Redact`
   - Calls `new AsyncConsoleAuditLogger().logAsync(...).join()`

---

## Side-by-Side Core Changes

| Component               | v1 (Java 8)                                         | v2 (Java 21)                                                     |
|-------------------------|-----------------------------------------------------|------------------------------------------------------------------|
| **Event Model**         | Mutable POJO                                        | Immutable `record`                                               |
| **Context**             | SLF4J `MDC`                                         | `ScopedValue`                                                    |
| **Logging**             | Synchronous `System.out`                            | Async virtual-thread executor                                    |
| **Redaction**           | Key-based filter (`password`)                      | Annotation-driven (`@Redact`) via reflection                     |
| **Build Config**        | `<source>1.8</source>`, version `1.0.0`              | `<source>21</source>`, version `2.0.0`                            |

---
📁 enterprise-audit-ecosystem/

── 📁 enterprise-audit-logger/
   ├── 📁 v1/
   │   ├── pom.xml
   │   └── src/
   │       └── main/java/com/enterprise/audit/
   │           ├── api/
   │           │   ├── AuditEvent.java
   │           │   └── AuditLogger.java
   │           ├── core/
   │           │   └── SimpleAuditLogger.java
   │           ├── redaction/
   │           │   └── Redactor.java
   │           └── utils/
   │               └── MDCUtil.java
   └── 📁 v2/
       ├── pom.xml
       └── src/
           └── main/java/com/enterprise/audit/
               ├── api/
               │   ├── AuditEvent.java
               │   ├── AuditLogger.java
               │   └── AuditContext.java
               ├── annotations/
               │   └── Redact.java
               ├── core/
               │   └── AsyncConsoleAuditLogger.java
               └── redaction/
                   └── Redactor.java

── 📁 audit-app-demo/
   ├── pom.xml
   └── src/
       └── main/java/com/enterprise/app/
           ├── MainAppV1.java
           └── MainAppV2.java


---

## enterprise-audit-logger/v1/pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.enterprise.audit</groupId>
  <artifactId>audit-logger</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>
  <properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>1.7.36</version>
    </dependency>
  </dependencies>
</project>
```

## enterprise-audit-logger/v1/src/main/java/com/enterprise/audit/api/AuditEvent.java
```java
package com.enterprise.audit.api;

import java.util.Map;

public class AuditEvent {
    private final String service;
    private final String user;
    private final String action;
    private final String result;
    private final long timestamp;
    private final Map<String, Object> metadata;

    public AuditEvent(String service, String user, String action, String result, Map<String, Object> metadata) {
        this.service = service;
        this.user = user;
        this.action = action;
        this.result = result;
        this.metadata = metadata;
        this.timestamp = System.currentTimeMillis();
    }

    public String getService() { return service; }
    public String getUser() { return user; }
    public String getAction() { return action; }
    public String getResult() { return result; }
    public long getTimestamp() { return timestamp; }
    public Map<String, Object> getMetadata() { return metadata; }

    @Override
    public String toString() {
        return "AuditEvent{" +
               "service='" + service + '\'' +
               ", user='" + user + '\'' +
               ", action='" + action + '\'' +
               ", result='" + result + '\'' +
               ", timestamp=" + timestamp +
               ", metadata=" + metadata +
               '}';
    }
}
```

## enterprise-audit-logger/v1/src/main/java/com/enterprise/audit/api/AuditLogger.java
```java
package com.enterprise.audit.api;

public interface AuditLogger {
    void log(AuditEvent event);
}
```

## enterprise-audit-logger/v1/src/main/java/com/enterprise/audit/core/SimpleAuditLogger.java
```java
package com.enterprise.audit.core;

import com.enterprise.audit.api.AuditEvent;
import com.enterprise.audit.api.AuditLogger;

public class SimpleAuditLogger implements AuditLogger {
    @Override
    public void log(AuditEvent event) {
        System.out.println("[AUDIT] " + event);
    }
}
```

## enterprise-audit-logger/v1/src/main/java/com/enterprise/audit/redaction/Redactor.java
```java
package com.enterprise.audit.redaction;

import java.util.Map;

public class Redactor {
    public static Map<String, Object> redact(Map<String, Object> metadata) {
        metadata.replaceAll((k, v) -> k.toLowerCase().contains("password") ? "[REDACTED]" : v);
        return metadata;
    }
}
```

## enterprise-audit-logger/v1/src/main/java/com/enterprise/audit/utils/MDCUtil.java
```java
package com.enterprise.audit.utils;

import org.slf4j.MDC;

public class MDCUtil {
    public static String getUser() {
        return MDC.get("user");
    }
    public static String getService() {
        return MDC.get("service");
    }
}
```

---

## enterprise-audit-logger/v2/pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.enterprise.audit</groupId>
  <artifactId>audit-logger</artifactId>
  <version>2.0.0</version>
  <packaging>jar</packaging>
  <properties>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>1.7.36</version>
    </dependency>
  </dependencies>
</project>
```

## enterprise-audit-logger/v2/src/main/java/com/enterprise/audit/api/AuditEvent.java
```java
package com.enterprise.audit.api;

import java.time.Instant;
import java.util.Map;

public record AuditEvent(
    String service,
    String user,
    String action,
    String result,
    Instant timestamp,
    Map<String, Object> metadata
) {}
```

## enterprise-audit-logger/v2/src/main/java/com/enterprise/audit/api/AuditLogger.java
```java
package com.enterprise.audit.api;

import java.util.concurrent.CompletableFuture;

public interface AuditLogger {
    void log(AuditEvent event);
    CompletableFuture<Void> logAsync(AuditEvent event);
}
```

## enterprise-audit-logger/v2/src/main/java/com/enterprise/audit/api/AuditContext.java
```java
package com.enterprise.audit.api;

public class AuditContext {
    public static final ScopedValue<String> USER = ScopedValue.newInstance();
    public static final ScopedValue<String> SERVICE = ScopedValue.newInstance();
}
```

## enterprise-audit-logger/v2/src/main/java/com/enterprise/audit/annotations/Redact.java
```java
package com.enterprise.audit.annotations;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
public @interface Redact {
    boolean enabled() default true;
}
```

## enterprise-audit-logger/v2/src/main/java/com/enterprise/audit/core/AsyncConsoleAuditLogger.java
```java
package com.enterprise.audit.core;

import com.enterprise.audit.api.AuditEvent;
import com.enterprise.audit.api.AuditLogger;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executors;
import java.util.concurrent.ExecutorService;

public class AsyncConsoleAuditLogger implements AuditLogger {
    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

    @Override
    public void log(AuditEvent event) {
        System.out.println("[AUDIT] " + event);
    }

    @Override
    public CompletableFuture<Void> logAsync(AuditEvent event) {
        return CompletableFuture.runAsync(() -> log(event), executor);
    }
}
```

## enterprise-audit-logger/v2/src/main/java/com/enterprise/audit/redaction/Redactor.java
```java
package com.enterprise.audit.redaction;

import com.enterprise.audit.annotations.Redact;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class Redactor {
    public static Map<String, Object> redact(Object metadataObject) {
        Map<String, Object> result = new HashMap<>();
        for (Field field : metadataObject.getClass().getDeclaredFields()) {
            field.setAccessible(true);
            Object value = null;
            try { value = field.get(metadataObject); } catch (IllegalAccessException ignored) {}
            boolean isSensitive = field.isAnnotationPresent(Redact.class);
            result.put(field.getName(), isSensitive ? "[REDACTED]" : value);
        }
        return result;
    }
}
```

---

## audit-app-demo/pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.enterprise.app</groupId>
  <artifactId>audit-app-demo</artifactId>
  <version>1.0-SNAPSHOT</version>

  <properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler-target>
  </properties>

  <dependencies>
    <!-- For v1 run, use version 1.0.0 -->
    <dependency>
      <groupId>com.enterprise.audit</groupId>
      <artifactId>audit-logger</artifactId>
      <version>1.0.0</version>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>1.7.36</version>
    </dependency>
  </dependencies>
</project>
```

## audit-app-demo/src/main/java/com/enterprise/app/MainAppV1.java
```java
package com.enterprise.app;

import com.enterprise.audit.api.AuditEvent;
import com.enterprise.audit.api.AuditLogger;
import com.enterprise.audit.core.SimpleAuditLogger;
import com.enterprise.audit.redaction.Redactor;
import com.enterprise.audit.utils.MDCUtil;
import org.slf4j.MDC;

import java.util.HashMap;
import java.util.Map;

public class MainAppV1 {
    public static void main(String[] args) {
        MDC.put("user", "alice");
        MDC.put("service", "order-service");

        AuditLogger logger = new SimpleAuditLogger();
        Map<String, Object> metadata = new HashMap<>();
        metadata.put("orderId", "ORD1234");
        metadata.put("password", "secret123");

        metadata = Redactor.redact(metadata);

        AuditEvent event = new AuditEvent(
                MDCUtil.getService(),
                MDCUtil.getUser(),
                "order.cancel",
                "SUCCESS",
                metadata
        );

        logger.log(event);
    }
}
```

## audit-app-demo/src/main/java/com/enterprise/app/MainAppV2.java
```java
package com.enterprise.app;

import com.enterprise.audit.api.AuditEvent;
import com.enterprise.audit.api.AuditLogger;
import com.enterprise.audit.api.AuditContext;
import com.enterprise.audit.core.AsyncConsoleAuditLogger;
import com.enterprise.audit.redaction.Redactor;
import com.enterprise.audit.annotations.Redact;

import java.time.Instant;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

public class MainAppV2 {
    static class Metadata {
        String orderId = "ORD5678";
        @Redact
        String password = "secret123";
    }

    public static void main(String[] args) {
        ScopedValue.where(AuditContext.USER, "bob")
                   .where(AuditContext.SERVICE, "billing-service")
                   .run(() -> {
                       AuditLogger logger = new AsyncConsoleAuditLogger();
                       Map<String, Object> metadata = Redactor.redact(new Metadata());

                       AuditEvent event = new AuditEvent(
                               AuditContext.SERVICE.get(),
                               AuditContext.USER.get(),
                               "billing.charge",
                               "SUCCESS",
                               Instant.now(),
                               metadata
                       );

                       CompletableFuture<Void> result = logger.logAsync(event);
                       result.join();
                   });
    }
}
```

---

## Side-by-Side Comparison of Core Changes

| Component                    | v1 (Java 8)                                                | v2 (Java 21)                                                    |
|------------------------------|------------------------------------------------------------|-----------------------------------------------------------------|
| **AuditEvent**               | POJO with fields, getters, `toString()`                    | `record` with fields                                            |
| **AuditLogger**              | `void log(AuditEvent)`                                     | `void log(AuditEvent)`, `CompletableFuture<Void> logAsync(...)` |
| **Context Propagation**      | SLF4J `MDC` via `MDCUtil.get*()`                          | `ScopedValue` in `AuditContext`                                 |
| **Logger Impl**              | `SimpleAuditLogger` synchronous `System.out`               | `AsyncConsoleAuditLogger` with virtual threads, async logging   |
| **Redaction**                | Map key filter (`password` key)                            | Annotation-driven (`@Redact`) via reflection on metadata class  |
| **App Usage**                | `MDC.put(...)`, manual `Map<String,Object>`                | `ScopedValue.where(...)`, metadata class                         |
| **Build (pom.xml)**          | Java 8 (`<source>1.8`) , version `1.0.0`                   | Java 21 (`<source>21`), version `2.0.0`                         |

---

