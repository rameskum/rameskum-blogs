---
title: 'Jackson 3 Is the Migration Inside Your Spring Boot 4 Migration'
date: 2026-10-10T11:00:00-04:00
draft: false
ShowToc: true
description: >-
  Spring Boot 4 quietly switched the default JSON engine to Jackson 3: new
  tools.jackson namespace, JsonMapper replacing ObjectMapper, immutable mappers,
  and unchecked exceptions. Your old Jackson 2 config bean silently stops
  applying. This is the before/after migration guide — namespace, builder API,
  custom serializers, and the order to do it in.
tags:
  - spring-boot
  - java
  - jackson
  - backend
  - interview
categories: article
keywords:
  - jackson 3 migration
  - tools.jackson namespace
  - JsonMapper vs ObjectMapper
  - spring boot 4 jackson 3
  - jackson 3 custom serializer
---

You migrated to Spring Boot 4. The app boots. Then a `RestClient` call starts failing on unknown properties — even though your `ObjectMapper` config disables `FAIL_ON_UNKNOWN_PROPERTIES`. You stare at the config bean. It's right there. It's also completely ignored.

Welcome to the migration inside the migration: **Spring Boot 4 switched the default JSON engine to Jackson 3**, and Jackson 3 is a new namespace, a new mapper, and new rules. Your Jackson 2 config doesn't error — it just silently stops applying. This guide is the before/after for the whole move.

## The namespace move: `tools.jackson`

Jackson 3 lives under a new Maven group and Java package:

```xml
<!-- before (Jackson 2) -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>

<!-- after (Jackson 3) -->
<dependency>
    <groupId>tools.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

```java
// before
import com.fasterxml.jackson.databind.ObjectMapper;
// after
import tools.jackson.databind.json.JsonMapper;
```

Two consequences worth knowing:

1. **Jackson 2 and 3 can coexist on the classpath.** The rename was deliberate — parallel artifacts mean you can migrate module by module instead of in one terrifying commit.
2. **Annotations are the exception.** `jackson-annotations` stays under `com.fasterxml.jackson.core`, so your `@JsonProperty` / `@JsonIgnore` imports on domain models don't change. (Caveat: databind's own annotations like `@JsonSerialize`/`@JsonDeserialize` *did* move to `tools.jackson.databind.annotation`.)

Baseline is Java 17, and 3.1.x is the first LTS in the 3.x line — if you're migrating now, target 3.1+, not the transitional 3.0.x.

## `JsonMapper` replaces `ObjectMapper` — and mappers are immutable now

In Jackson 3, `ObjectMapper` still exists but it's the format-agnostic base; for JSON you use `JsonMapper`, and **all mappers are immutable** — configuration happens through builders, not setters:

```java
// before (Jackson 2)
ObjectMapper mapper = new ObjectMapper();
mapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
mapper.registerModule(new JavaTimeModule());

// after (Jackson 3)
JsonMapper mapper = JsonMapper.builder()
        .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
        .addModule(new JavaTimeModule())
        .build();
```

Jackson 3 mappers have no setters — configure them only through the builder. That also removes a whole class of "who mutated the shared mapper" bugs, because the shared instance can't be changed after `build()`.

Format-specific mappers are now mandatory: `JsonMapper`, `YAMLMapper`, `XmlMapper`. The old `new ObjectMapper(new YAMLFactory())` pattern is gone — if you were constructing mappers per format, that's a compile error now, which is the good kind of breakage.

## The silent killer: your Jackson 2 config bean stops applying

This is the 3 AM one from the intro. Spring Boot 4 auto-configures a Jackson 3 `JsonMapper` for its HTTP message converters — including the ones inside `RestClient`. If your config class still builds a Jackson 2 `ObjectMapper`:

```java
// This bean is now invisible to Boot 4's converters
@Bean
public ObjectMapper objectMapper() {   // com.fasterxml.jackson version
    // ...
}
```

Boot sees a `com.fasterxml.jackson` bean, shrugs, and configures its own `tools.jackson` `JsonMapper` with defaults. Your `FAIL_ON_UNKNOWN_PROPERTIES` never reaches the `RestClient`. The fix is mechanical but must be complete:

```java
import tools.jackson.databind.json.JsonMapper;
import tools.jackson.databind.DeserializationFeature;

@Bean
public JsonMapper jsonMapper() {
    return JsonMapper.builder()
            .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
            .build();
}
```

**Migration rule: grep for `com.fasterxml.jackson.databind` (not `annotation`) — every hit is a migration task.** The annotations package staying put is what makes a partial grep dangerous; `com.fasterxml.jackson.annotation` imports are fine, `com.fasterxml.jackson.databind` imports are not.

## Custom serializers: new base classes, unchecked exceptions

If you write custom serializers/deserializers, the base classes moved and the exception model changed:

```java
// before (Jackson 2)
public class MoneySerializer extends JsonSerializer<Money> {
    public void serialize(Money v, JsonGenerator gen,
                          SerializerProvider serializers) throws IOException {
        gen.writeString(v.toString());
    }
}

// after (Jackson 3)
public class MoneySerializer extends ValueSerializer<Money> {
    public void serialize(Money v, JsonGenerator gen,
                          SerializationContext context) {
        gen.writeString(v.toString()); // JacksonException is unchecked now
    }
}
```

Three things changed at once: `JsonSerializer` → `ValueSerializer` (and `JsonDeserializer` → `ValueDeserializer`), the context parameter type, and **checked `IOException` is gone** — Jackson 3 throws unchecked `JacksonException`, so `throws` declarations and try/catch scaffolding around mapper calls can go. In production, watch enums: Jackson 3 no longer consults `findSerializer` for them, so register enum handling via `findEnumSerializer`. The tree API shifted too — `readTree` via the context, `asString()`, `writeName()`.

Feature flags got renamed too: `JsonParser.Feature` split into `JsonReadFeature` and `StreamReadFeature`. If you toggled parser features, check each one against the new names.

## The migration order for a real codebase

1. **Bump to Jackson 3.1.x** via the Boot 4 BOM — don't hand-manage versions.
2. **Fix imports mechanically**: `com.fasterxml.jackson.databind` → `tools.jackson`, keeping `com.fasterxml.jackson.annotation` as-is. Your IDE's replace-in-path does 80% of this.
3. **Convert mapper construction to builders.** Every `new ObjectMapper()` + setter chain becomes `JsonMapper.builder()...build()`. Every `new ObjectMapper(new YAMLFactory())` becomes `new YAMLMapper()`.
4. **Audit config beans.** Any `@Bean` producing a Jackson 2 type must become a `JsonMapper` bean, or Boot 4 silently ignores it. This is the step that causes production incidents — do it before you declare victory.
5. **Port custom serializers** to `ValueSerializer`/`ValueDeserializer`, drop `throws IOException`, fix enum handling.
6. **Run the full suite and diff your JSON.** Changed defaults plus renamed features mean output can shift subtly; snapshot-test your critical payloads.

## The 2-minute interview answer

> "Spring Boot 4 moved to Jackson 3, which is a new namespace — `tools.jackson` instead of `com.fasterxml.jackson` — so 2.x and 3.x can coexist during migration. `JsonMapper` replaces `ObjectMapper` for JSON, mappers are immutable and built with builders, and exceptions are unchecked now. The nasty part: Boot 4 auto-configures a Jackson 3 `JsonMapper` for its converters, so a leftover Jackson 2 `ObjectMapper` bean is silently ignored — your config stops applying with no error. Annotations stay under the old package, which is why you grep for `databind` imports specifically. Custom serializers move to `ValueSerializer`/`ValueDeserializer`, and format-specific mappers like `YAMLMapper` are mandatory."

## Cheat sheet

| Before (Jackson 2) | After (Jackson 3) |
|---|---|
| `com.fasterxml.jackson.*` | `tools.jackson.*` (annotations stay put) |
| `new ObjectMapper()` + setters | `JsonMapper.builder()...build()` — immutable |
| `new ObjectMapper(new YAMLFactory())` | `new YAMLMapper()` — format mappers mandatory |
| `@Bean ObjectMapper` | `@Bean JsonMapper` — or Boot 4 ignores it silently |
| `JsonSerializer` / `JsonDeserializer` | `ValueSerializer` / `ValueDeserializer` |
| `throws IOException` | Unchecked `JacksonException` — drop the scaffolding |
| `JsonParser.Feature` | `JsonReadFeature` / `StreamReadFeature` |
| Enum via `findSerializer` | `findEnumSerializer` — the old hook isn't consulted |

The migration is mechanical, which is exactly why it's dangerous: everything *looks* fine until a config bean silently stops applying at runtime. Grep for `databind`, convert the beans, diff your JSON — in that order.
