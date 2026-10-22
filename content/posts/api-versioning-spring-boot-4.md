---
title: 'API Versioning in Spring Boot 4: The Built-In Way'
date: 2026-10-19T13:30:00+05:30
draft: false
ShowToc: true
description: >-
  Spring Framework 7 adds a version attribute to @RequestMapping and every
  shortcut variant — first-class API versioning with pluggable resolution
  strategies, baseline versions, and deprecation handling. How to retire the
  hand-rolled /v1 prefixes, and the two defaults that break existing clients
  if you miss them.
tags:
  - spring-boot
  - java
  - backend
  - api
categories: article
keywords:
  - spring boot 4 api versioning
  - spring framework 7 version attribute
  - api versioning requestmapping
  - rest api versioning spring
---

Every Spring codebase has its own API versioning story, and they're all slightly embarrassing: `/v1` and `/v2` prefixes baked into paths, a custom header parsed by hand in an interceptor, or two parallel controller classes that drift out of sync. Spring Framework 7 — the foundation of Spring Boot 4 — finally makes versioning a first-class citizen: a `version` attribute on `@RequestMapping` and every shortcut variant, with the framework handling resolution, matching, and deprecation. Here's how to use it, and where it bites.

## The old ways (and why they hurt)

Before Framework 7, request mapping had no concept of a version at all. You bolted it on from the outside:

- **Path prefixes** (`/api/v1/users` vs `/api/v2/users`): works, but the version pollutes every URL, clients hardcode it, and you duplicate controller classes.
- **Custom headers with manual parsing**: an interceptor or a shared dispatcher method reads `X-API-Version` and branches with `if` statements — routing logic hiding in business code.
- **Media type parameters** (`Accept: application/json;version=2`): elegant in theory, painful to test and debug.

The built-in model replaces all three with one mechanism: the version becomes part of the mapping, exactly like the path and the HTTP method.

## The new model: three moving parts

```java
@RestController
@RequestMapping("/accounts")
public class AccountController {

    @GetMapping(path = "/{id}", version = "1.0")
    public AccountV1 getAccountV1(@PathVariable Long id) {
        return accountService.findV1(id);
    }

    @GetMapping(path = "/{id}", version = "2.0")
    public AccountV2 getAccountV2(@PathVariable Long id) {
        return accountService.findV2(id);
    }
}
```

Same path, two versions, two methods. A request for version `1.0` hits the first; `2.0` hits the second. No `if` checks, no manual parsing, no shared dispatcher.

Three components make it work:

1. **ApiVersionResolver** — extracts the raw version from the request (header, query param, path segment, or media type parameter).
2. **ApiVersionParser** — turns that raw string into a comparable version. The built-in `SemanticApiVersionParser` understands `major.minor.patch`.
3. **ApiVersionStrategy** — matches the parsed version against the `version` declared on each mapping and picks the closest handler. If the requested version isn't supported, it throws `InvalidApiVersionException`.

## Configuring: pick one strategy

You configure resolution through `WebMvcConfigurer`'s `ApiVersionConfigurer`. A correction to a claim you'll see repeated (including in early drafts of this post): the strategies *can* technically be combined — each `use*` call adds a resolver, and the strategy uses the first one that produces a version. Nothing in the reference docs forbids mixing them. The advice is a team convention, not a framework rule — and it's still the right convention: pick one.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureApiVersioning(ApiVersionConfigurer configurer) {
        configurer
            .useRequestHeader("X-API-Version")   // header-based
            // .useQueryParam("version")        // ?version=2.0
            // .usePathSegment(1)              // /api/v1/users
            // .useMediaTypeParameter(MediaType.APPLICATION_JSON, "version")
            .addSupportedVersions("1.0", "2.0");
    }
}
```

Or properties — `spring.mvc.apiversion.use.header=X-API-Version` picks the header strategy without the configurer. The header strategy (`X-API-Version`) is the most common choice for internal and partner APIs because it keeps URLs stable — but the real rule is simpler: **pick one and use it everywhere**. Mixed strategies across services are worse than any single strategy.

## Two defaults that will bite you

**A version is required by default.** Once versioning is enabled, a request carrying no version fails with `MissingApiVersionException` — HTTP 400. That includes every existing client that doesn't send `X-API-Version` yet: they break the moment you turn this on. To keep them working, set a default version (`setDefaultVersion("1.0")` — setting one also makes the version optional), or call `setVersionRequired(false)` without a default, in which case version-less requests are routed to the most recent supported version. Choose deliberately: "latest" as an accidental fallback during a migration is how clients silently change behavior.

**Supported versions are detected for you.** By default, the strategy builds the supported set from the versions declared on your mappings, so the `.addSupportedVersions("1.0", "2.0")` above is belt-and-braces, not a requirement. The explicit list matters when you turn detection off (`detectSupportedVersions(false)`) — worth doing on public APIs, so a typo'd `version = "20"` on some mapping doesn't silently become a supported version.

## Baseline versions: don't copy unchanged endpoints

The quiet killer of hand-rolled versioning is endpoint duplication: v2 ships with three changed endpoints, so you copy all forty. The built-in model has baseline versions to kill that:

```java
@GetMapping(path = "/{id}", version = "2.0+")
public AccountV2 getAccount(@PathVariable Long id) { ... }
```

`"2.0+"` is a baseline: the mapping handles version 2.0 **and every later version**, unless a more specific mapping exists for a newer version. Unchanged endpoints get one mapping with a baseline; only the endpoints that actually changed get per-version handlers. When v3 arrives, you add handlers only where behavior changed. This is the single biggest maintenance win of the built-in model.

## Deprecation: headers, not surprises

Versions die, and the framework has opinions about how. The built-in implementation is `StandardApiVersionDeprecationHandler` (in `org.springframework.web.accept`): it sends standard deprecation signals — `Deprecation` and `Sunset` headers per RFC 9745 and RFC 8594, plus a `Link` header pointing at your migration docs — so clients learn a version is dying from the responses themselves, not from a Slack message:

```java
@Override
public void configureApiVersioning(ApiVersionConfigurer configurer) {
    StandardApiVersionDeprecationHandler deprecationHandler = new StandardApiVersionDeprecationHandler();
    deprecationHandler.configureVersion("1.0")
        .setDeprecationDate(ZonedDateTime.parse("2026-06-01T00:00:00Z"))
        .setSunsetDate(ZonedDateTime.parse("2026-12-01T00:00:00Z"))
        .setSunsetLink(URI.create("https://api.example.com/docs/migrate-to-2.0"));

    configurer.useRequestHeader("X-API-Version")
        .setDeprecationHandler(deprecationHandler);
}
```

Deprecate with headers and dates, never with surprise removals: announce the sunset date in the response, keep serving the old version until it, then remove it.

## The client side: ApiVersionInserter

Versioning is a contract, and the client half of the contract needs to be as boring as the server half. Spring's `RestClient` supports an `ApiVersionInserter` so outgoing calls set the version the same way on every request — no hand-rolled header code in every service client, no forgotten version on the one call that matters:

```java
RestClient restClient = RestClient.builder()
    .baseUrl("https://api.example.com")
    .apiVersionInserter(ApiVersionInserter.useHeader("X-API-Version"))
    .build();

// per request — the inserter puts the version where the strategy expects it
Account account = restClient.get()
    .uri("/accounts/{id}", 42)
    .apiVersion("2.0")
    .retrieve()
    .body(Account.class);
```

The same support exists across the client family: `WebClient` takes an inserter the same way, HTTP Service clients can declare the version right on the interface method (`@GetExchange(version = "2.0")`), and the test clients — MockMvc and `WebTestClient`/`RestTestClient` — accept an inserter too, so tests version their requests exactly the way production clients do.

## A note on OpenAPI docs

Shorter and more honest than a "gotcha": the standard docs tool for Boot is springdoc-openapi, and I have not run a versioned Boot 4 app through it to capture what its generated spec does with two handlers sharing one path — so this is a verification step for your stack, not a confirmed defect. When you generate the spec, check three things: both versions' operations appear for the shared path, the version header shows up as a parameter, and the per-version schemas aren't merged into one. If your springdoc version flattens them, group the docs per version until it catches up. Thirty seconds with the generated JSON tells you more than any blog post — including this one.

## Where this fits the migration

If you're on the Boot 4 migration path, this is a "while you're here" improvement, not a migration requirement — the old path-prefix controllers keep working. But it's the right moment: you're already touching controllers for the Jackson 3 and starter changes, and converting `/v1`-prefixed paths to the built-in model is a mechanical change with a real maintenance payoff. One rollout caveat, from the defaults above: the moment you enable the built-in model, version-less requests to the new endpoints start returning 400 — set a default version before you ship, or the existing clients that don't send the header break on day one. Don't mix it into the same PR as the framework upgrade, though — one concern per change, especially during a migration.

## The 2-minute interview answer

> "Spring Framework 7 added a `version` attribute to `@RequestMapping` and all its variants, so API versioning is built into request mapping. Three parts: a resolver extracts the version from the request — header, query param, path segment, or media type, configured once via `ApiVersionConfigurer` — a parser turns it into a comparable semantic version, and a strategy picks the closest matching handler, throwing for unsupported versions. Baseline versions like `"2.0+"` handle a version and everything after it, so unchanged endpoints aren't duplicated. Deprecation goes through standard `Deprecation`/`Sunset` headers from the built-in handler, clients set versions with `ApiVersionInserter`, and two defaults matter: a version is required unless you set a default (version-less requests get a 400), and supported versions are auto-detected from your mappings. One verification step remains yours: check the generated OpenAPI spec shows each version, because doc tooling thinks in paths, not path-plus-version."

## Cheat sheet

| Piece | Detail |
|---|---|
| Annotation | `version` on `@RequestMapping` / `@GetMapping` / `@PostMapping` / … |
| Strategies | Header, query param, path segment, media type — combinable in the framework, but pick **one** as a team convention |
| Config | `WebMvcConfigurer.configureApiVersioning` or `spring.mvc.apiversion.*` |
| Missing version | 400 by default (`MissingApiVersionException`) — set a default version, or make it optional (falls back to latest) |
| Supported versions | Auto-detected from mappings by default; `addSupportedVersions` needed only when detection is off |
| Baseline | `"2.0+"` handles 2.0 and later unless a more specific mapping exists |
| Unsupported version | `InvalidApiVersionException` |
| Deprecation | `Deprecation` / `Sunset` / `Link` headers via `StandardApiVersionDeprecationHandler` (RFC 9745 / 8594) |
| Client | `ApiVersionInserter` on `RestClient` / `WebClient`; version also declarable on HTTP Service clients |
| Docs check | Generate the OpenAPI spec and confirm each version appears — not verified against springdoc here, check yours |
| Migration note | Old `/v1` prefixes keep working; convert separately, one concern per change |

The version is part of the mapping now. Use it like one.
