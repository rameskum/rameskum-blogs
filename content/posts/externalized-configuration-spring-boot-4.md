---
title: 'Externalized Configuration in Spring Boot 4: Change a Property Without Restarting the Service'
date: 2026-10-07T08:00:00-04:00
draft: false
ShowToc: true
description: >-
  Every microservice team eventually learns the hard way: a timeout, a feature
  flag, or a retry count baked into application.yml means a rebuild and a
  rolling restart to change one value. Externalized configuration fixes that —
  one Git-backed Config Server, profiles per environment, refresh without
  restart, and a git history that answers "who changed this property?" Full
  Spring Boot 4 + Spring Cloud Config working setup with docker-compose.
tags:
  - spring-boot
  - java
  - microservices
  - backend
  - system-design
  - interview
categories: article
keywords:
  - spring cloud config server
  - externalized configuration spring boot
  - spring boot refresh scope
  - spring cloud bus refresh
  - microservice configuration management
---

It's 3 AM and the payment provider is timing out. The fix is one property: raise `payments.timeout` from 5s to 15s until their incident clears. But that property lives in `application.yml`, baked into the jar, deployed to twelve instances. So you rebuild, push, and roll twelve restarts — for a value that isn't code.

Externalized configuration is the pattern that ends this ritual: configuration lives outside the deployable artifact, in one versioned place, and services pick it up at startup — or at runtime, without restarting. Spring Cloud Config is the canonical Spring implementation, and it pairs with a Boot 4 idiom (config-data import) that killed the old `bootstrap.yml` dance years ago.

## The pieces

Three moving parts, each boring on its own:

1. **Config Server** — a tiny Spring Boot app (`spring-cloud-config-server`) that serves configuration over HTTP from a Git repo (or Vault, or the local filesystem).
2. **Config clients** — your services, each with `spring-cloud-starter-config` and one line in `application.yml` pointing at the server.
3. **Git as the source of truth** — every property change is a commit, which means it has an author, a timestamp, and a diff.

On Boot 4 you pair this with Spring Cloud **2025.1.x (Oakwood)** — the release train for Boot 4.0.x — via the `spring-cloud-dependencies` BOM. Mix a 2025.0.x (Boot 3.5) train with Boot 4 and you'll get exactly the classpath misery the BOM exists to prevent.

## Build it: Config Server in ten minutes

The server is a whole Spring Boot application in about 30 lines. The BOM import is the part that matters:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2025.1.2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
</dependencies>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# config-server's own application.yml
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/service-config.git
          default-label: main
          # clone on startup so the first client never waits on a cold clone
          clone-on-start: true
```

That's a working server. It exposes `{application}/{profile}[/{label}]` — e.g. `GET /payments-service/prod` returns the merged property sources for that service and profile as JSON. Try it with curl before wiring any client; debugging config through a running service is miserable, debugging it against the server's HTTP API is pleasant.

Don't have a config repo yet? For local development, switch the backend to the native filesystem profile instead of Git:

```yaml
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: file:./config-repo
```

Same HTTP API, no repo required. Promote to Git when the team grows past one laptop.

## The client: one line, no bootstrap

If you learned Spring Cloud before 2021, you remember `bootstrap.yml` and a separate bootstrap context. That's legacy. Since Boot 2.4, the modern way is config-data import — remote config loads as a first-class part of normal `application.yml` processing:

```yaml
# payments-service's application.yml
spring:
  application:
    name: payments-service
  config:
    import: "optional:configserver:http://config-server:8888"
```

Two details that bite people:

- `optional:` means "don't crash at startup if the server is unreachable." Drop it in production — if your service can't boot without its config, you want to fail loudly at startup, not boot with defaults and discover it at 3 AM. Pair with `spring.cloud.config.fail-fast: true` and retry settings so a slow server doesn't kill the boot either.
- `spring.application.name` is what the server uses to find `{name}.yml` / `{name}-{profile}.yml` in the repo. Get the name wrong and you silently get defaults — the most common "config server isn't working" support ticket.

The full pattern in `docker-compose.yml`, server plus one client:

```yaml
services:
  config-server:
    build: ./config-server
    ports:
      - "8888:8888"
    environment:
      - SPRING_CLOUD_CONFIG_SERVER_GIT_URI=https://github.com/your-org/service-config.git

  payments-service:
    build: ./payments-service
    depends_on:
      - config-server
    environment:
      - SPRING_PROFILES_ACTIVE=prod
```

## Bind it properly: @ConfigurationProperties, not @Value soup

Fetching config is half the job; binding it to something reviewable is the other half. One `@ConfigurationProperties` class beats twenty `@Value` annotations scattered across services — it's typed, it's validatable, and `/actuator/configprops` shows you the live binding:

```java
@Component
@ConfigurationProperties(prefix = "payments")
@Validated
public class PaymentsProperties {

    @NotNull
    @Min(1)
    private Duration timeout = Duration.ofSeconds(5);

    @Min(0)
    @Max(5)
    private int retryAttempts = 3;

    private boolean providerFallbackEnabled = false;

    // getters and setters
}
```

Now the 3 AM scenario is a commit to `payments-service-prod.yml`:

```yaml
payments:
  timeout: 15s
  retry-attempts: 3
```

Validation matters here more than usual: a typo'd `timeout: -5s` failing fast at startup with a `BindValidationException` is infinitely better than a service running with a nonsense value. Fail at boot, not at 3 AM.

## Refresh without restart

Changed the property in Git — now what? Without refresh, the answer is still "restart the service." Two levels of fix:

**Level 1: one instance.** Annotate the bean with `@RefreshScope` and POST to `/actuator/refresh`:

```java
@RestController
@RefreshScope
public class PaymentsController {
    private final PaymentsProperties props;
    // ...
}
```

```bash
curl -X POST http://localhost:8080/actuator/refresh
```

`@RefreshScope` recreates the bean on refresh, re-binding the new values. Caveat: it only works on beans where re-creation is safe — don't put it on a connection pool or anything holding state. And `@ConfigurationProperties` beans re-bind on refresh even without `@RefreshScope` in current versions; the annotation is for `@Value`-injected or otherwise cached values. Expose the endpoint first: `management.endpoints.web.exposure.include=refresh`.

**Level 2: twelve instances.** POSTing refresh to each instance is the restart dance with extra steps. Spring Cloud Bus broadcasts the refresh over a message broker — one POST, every instance updates:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

```bash
curl -X POST http://localhost:8080/actuator/busrefresh
```

One call fans out to every bus member. If you already run Kafka (and if you're reading this blog's Kafka posts, you do), the bus is nearly free.

## Secrets: the thing that doesn't go in Git

Everything above assumes configuration. Secrets are not configuration. API keys, database passwords, and tokens in a Git repo — even a private one — is how credentials leak, because Git history is forever and repo access is broad.

The pattern: keep secrets out of the config repo entirely.

- **Simplest:** environment variables or your orchestrator's secret store. Spring's property precedence puts env vars above config-server values, so `PAYMENTS_API_KEY` overrides whatever the repo says. Compose makes this natural: `env_file` or Docker secrets.
- **When you outgrow that:** Config Server has a Vault backend (`spring.cloud.config.server.vault.*`), and there's a dedicated `spring-cloud-vault` client. The server fetches secrets from Vault at request time; they never touch Git.

The rule of thumb for the interview: config repo holds *non-sensitive* environment-specific values; secrets come from a secret store and override via property precedence. If someone has to ask "is this a secret?", treat it as one.

## "Who changed this property?" — config as audit trail

Here's the part nobody writes about, and the reason this pattern earns its keep in production support. Because every value is a Git commit:

- `git log -p -- payments-service-prod.yml` tells you exactly who changed `payments.timeout`, when, and what it was before. Compare that with "someone edited a value in a dashboard three weeks ago."
- The server's `GET /payments-service/prod` endpoint shows the *resolved* configuration with its property-source ordering — when a value isn't what you expect, the `propertySources` array shows you which file won.
- On the client, `/actuator/env` shows the effective environment and `/actuator/configprops` shows the live `@ConfigurationProperties` bindings. The standard "the service is behaving as if the old value is still there" debug loop is: check `/actuator/env` → is the new value present? If yes, the bean didn't refresh. If no, the server is serving stale config (check the label/branch).

Tag the config repo on every production deploy. Then "what was the config when this incident started?" is a `git show` away instead of a guess.

## The 2-minute interview answer

> "Externalized configuration means services don't bake environment-specific values into the artifact. I run a Spring Cloud Config Server backed by Git — one commit changes a property for every instance, with full history. Clients pull config at startup via `spring.config.import=configserver:...`, bind it with validated `@ConfigurationProperties`, and pick up changes at runtime with `@RefreshScope` plus Spring Cloud Bus instead of restarts. Secrets stay out of Git — env vars or Vault, overriding via property precedence. And because it's all versioned, 'who changed this property' is a git log away."

## Cheat sheet

| Task | How |
|---|---|
| Server setup | `spring-cloud-config-server` + `@EnableConfigServer`, BOM `2025.1.x` for Boot 4 |
| Git backend | `spring.cloud.config.server.git.uri`, `default-label`, `clone-on-start: true` |
| Local dev without Git | `spring.profiles.active=native` + `search-locations` |
| Client wiring (modern) | `spring.config.import: "optional:configserver:http://config-server:8888"` — no `bootstrap.yml` |
| Fail loud in prod | Drop `optional:`, set `spring.cloud.config.fail-fast: true` + retry |
| Type-safe binding | `@ConfigurationProperties` + `@Validated`; inspect via `/actuator/configprops` |
| Refresh one instance | `@RefreshScope` + `POST /actuator/refresh` |
| Refresh all instances | `spring-cloud-starter-bus-kafka` + `POST /actuator/busrefresh` |
| Secrets | Never in the config repo — env vars/Docker secrets, or Vault backend |
| Debug a wrong value | Server: `GET /{app}/{profile}` → `propertySources`; client: `/actuator/env` |
| Audit trail | `git log -p` on the config repo; tag it per deploy |

The 3 AM timeout change is now a one-line commit, a `busrefresh`, and back to sleep — no rebuild, no rolling restart, and a git history that proves what changed. That's the whole pattern: configuration is data with a lifecycle, not code with a build.
