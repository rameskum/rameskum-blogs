---
title: 'Solve Every Error Once: The Fleet Playbook for Migrating 60+ Services to Spring Boot 4'
date: 2026-10-18T12:00:00+05:30
draft: false
ShowToc: true
description: >-
  Migrating one service to Spring Boot 4 is a weekend. Migrating sixty is a
  program — and the failure mode is every team rediscovering the same seven
  errors. The fleet playbook: a shared chassis, an error catalog, OpenRewrite
  recipes that solve each error permanently, and the rollout loop.
tags:
  - spring-boot
  - java
  - backend
  - microservices
categories: article
keywords:
  - spring boot 4 migration fleet
  - openrewrite spring boot 4
  - spring boot upgrade at scale
  - spring boot 4 breaking changes
---

Migrating one service to Spring Boot 4 is a weekend project. Migrating sixty is a program — and programs fail in a specific way: every team independently rediscovers the same seven errors, burns a week each, and the "migration" becomes sixty snowflake upgrades with sixty subtly different outcomes.

The fix is a discipline with an unfriendly name and a friendly result: **solve every error once**. When a team hits a migration error, the output isn't a fixed service — it's a permanent, automated fix that no other team ever has to think about again. Here's the playbook.

## The shape of the program

1. **Chassis first.** One shared parent POM (or BOM + conventions) holding the Java version, dependency management, and test dependencies for the whole fleet. If every service declares its own dependency versions, you don't have a fleet — you have sixty independent migrations.
2. **Pilot with the hardest service**, not the easiest. Pick the one with the gnarliest dependencies, the most custom Jackson config, the weirdest test setup. Every error it surfaces is an error the easy services would have hit later, louder.
3. **Error catalog.** Every break the pilot hits gets a written entry: the symptom, the root cause, the fix. This catalog is the program's actual deliverable.
4. **Codify each error as an OpenRewrite recipe** (or a script, or a checklist step — but a recipe is better, because it runs itself).
5. **Roll out in waves**, recipe-first: each service gets the automated fixes applied, then human attention only for what's left.

The loop that makes it work: when wave two surfaces a *new* error, it goes into the catalog, becomes a recipe, and waves three through sixty never see it. The error count per service should decay to near zero by the final wave. If it doesn't, your catalog is leaking.

## The seven errors (from a real migration)

These are the seven breaks from Dan Vega's documented migration, ["Spring Boot 3 to 4 Migration: Fixing Every Break"](https://www.danvega.dev/blog/spring-boot-3-to-4-migration), which took a working app from Spring Boot 3.5.16 to 4.1 and fixed each break as it surfaced. They're representative of what the whole fleet will hit, in roughly this order — compiler errors first, silent behavior changes last:

**1. Jackson 2 → 3.** Imports move `com.fasterxml.jackson` → `tools.jackson`, and `ObjectMapper` gives way to the immutable `JsonMapper` builder. The compiler catches most of it. (The full story is its own guide — the namespace move, the silent `@Bean` problem, custom serializers.)

**2. Renamed starters and test APIs.** `spring-boot-starter-web` → `spring-boot-starter-webmvc`, test imports move packages, `@MockBean`/`@SpyBean` are removed in favor of `@MockitoBean`/`@MockitoSpyBean`. Also compiler-caught — mostly.

**3. Flyway silently never runs.** Without `spring-boot-starter-flyway`, migrations don't execute and nothing tells you. Your tests catch this *if* they assert on schema state. Many don't.

**4. Hand-built Jackson mappers change date format.** A manually constructed mapper now writes ISO-8601 dates instead of timestamps. Tests catch it if they assert on serialized output.

**5. The H2 console vanishes.** It needs `spring-boot-h2console` now. Silent until someone opens the console URL.

**6. JSpecify nullness.** Spring's JSR-305 annotations give way to JSpecify's `@NullMarked` and `@Nullable`. Builds with null checkers (or Kotlin) can fail on code nobody touched.

**7. Jackson 3 rejects null into primitives.** A client sending `null` for a primitive field now gets a 400. No warning at startup — it shows up as a changed API contract under real traffic.

The pattern: the compiler catches 1–2, tests catch 3–4 *if they exist*, and 5–7 are silent. That's why "green build" is not the migration's done criteria — after the build is green, start the app, click around, send real requests. The silent changes only show up under real traffic.

Two more to add to your catalog from the broader migration guides: **Undertow is removed outright** (no Servlet 6.1 support — you're changing servers, not just versions; strip `server.undertow.*` properties), and the **Jackson 2 escape hatch** `spring.jackson.use-jackson2-defaults=true` exists as a *deprecated bridge* to buy time — never as the end state.

## OpenRewrite: the error catalog as executable code

This is where "solve every error once" becomes literal. OpenRewrite's Spring Boot 4 composite recipe chains the upgrade — it includes the intermediate migrations, the Spring Boot 3.5 migration among them. Even so, the recommended fleet path is one hop from a green latest-3.5.x baseline (see Prerequisites below), not a leap from wherever each service happens to sit:

```bash
# one-shot, on a dedicated branch, from a green latest-3.5.x baseline
mvn -U org.openrewrite.maven:rewrite-maven-plugin:6.46.1:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:6.37.1 \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot4.UpgradeSpringBoot_4_0 \
  -Drewrite.exportDatatables=true
```

Two reproducibility rules for a fleet run. First, pin everything: the command above pins `rewrite-maven-plugin` 6.46.1 and `rewrite-spring` 6.37.1 (the latest of each on Maven Central as of writing; both resolve with no extra repository) — never `RELEASE`. With `RELEASE`, every service's run resolves whatever is newest that day, and "solve every error once" quietly stops meaning the same fix everywhere. Hold both versions in the chassis and bump them deliberately, like any other fleet-wide change. Second, know your edition: this command runs the **Community Edition** recipe from the open-source `rewrite-spring` on Maven Central. OpenRewrite also publishes a Moderne Edition of the Boot 4 migration with a broader recipe set, and its docs reference newer `rewrite-spring` builds (6.41.0 at the time of writing) that were not on Maven Central when I checked — so if a recipe list you're reading doesn't match the Community Edition definition, or the version in a doc snippet fails to resolve, you're probably looking at the other edition.

Per that definition, the composite recipe (`org.openrewrite.java.spring.boot4.UpgradeSpringBoot_4_0`) chains the Spring Boot 3.5 migration, Spring Cloud 2025.1, Spring Framework 7, Spring Security 7, Spring Batch 6, Hibernate 7.1, the Boot 4 properties migration, the `@MockBean`/`@SpyBean` replacement, the modular-starter migration and starter renames, and the `AddAutoConfigureTestRestTemplate` family — each independently tested and versioned. One thing it does *not* include: a Java version upgrade. Boot 4's baseline is Java 17, and the composite leaves your Java version alone — if you want Java 25, chain `org.openrewrite.java.migrate.UpgradeToJava25` as a separate, deliberate step. There are also surgical recipes for individual errors, like `AddAutoConfigureTestRestTemplate` (the `TestRestTemplate` bean is no longer auto-configured in Boot 4) and `AddModularStarterDependencies` (adds the Boot 4 modular starter dependencies implied by your Boot 3 package usage, without relocating any source; `MigrateToModularStarters` is the wider recipe that also moves the code).

The fleet move: **write your own recipes for your catalog**. Your company's custom starter, your internal auth library, the weird Jackson module only you use — when the pilot hits it, encode the fix as a recipe in your own recipe module. Then every subsequent service gets `mvn rewrite:run` and the fix applies itself. The recipe module becomes a versioned artifact of institutional knowledge, and next year's upgrade starts from it instead of from zero.

Prerequisites, non-negotiable: standardize the fleet on the latest 3.5.x first, with a green build and passing tests, and do the work on a dedicated branch. Yes, the composite chains the intermediate migrations — the Boot 3.5 recipe is in its definition — but running it as one hop from a known-good 3.5.x baseline is the safer fleet process: when a service comes out wrong, you know the recipe run caused it, not some half-finished intermediate state each team reached differently. OpenRewrite rewrites files; you want the diff reviewable.

## Parallelizing without repeating work

- **Wave the rollout.** Pilot (1–2 hard services) → early wave (5–10 typical services) → the fleet. Each wave's new errors become recipes before the next wave starts.
- **One branch per service, CI-gated.** The recipe run is a commit; the existing test suite is the gate. A service merges when its pipeline is green *and* someone has clicked through the running app (remember the silent errors).
- **The catalog is a living doc with an owner.** New error → root cause → fix → recipe → announced to all teams. Without an owner it becomes a wiki page nobody reads.
- **Track error-per-service across waves.** That number decaying toward zero is how you know the program is working. If wave four hits as many new errors as wave one, something is wrong with the recipe pipeline, not the services.

## The 2-minute interview answer

> "Migrating a fleet to Spring Boot 4 is a program, not sixty upgrades. The failure mode is every team rediscovering the same errors. The playbook: a shared chassis holding versions and conventions, a pilot on the hardest service to build an error catalog, and every catalog entry codified as an OpenRewrite recipe so it's solved permanently — the error count per service should decay to zero across waves. The errors themselves: Jackson 2→3 and renamed starters are compiler-caught; Flyway and hand-built mappers are test-caught; the H2 console, JSpecify nullness, and Jackson 3's null-into-primitive 400s are silent and only show up under real traffic. OpenRewrite's Boot 4 composite recipe chains the intermediate upgrades automatically, and your own recipe module captures company-specific fixes as versioned knowledge for the next upgrade."

## Cheat sheet

| Step | Detail |
|---|---|
| Chassis | Shared parent/BOM: Java version, dependency management, test deps |
| Pilot | Hardest service first — its errors are everyone's errors |
| Catalog | Symptom → root cause → fix, owned and living |
| Codify | Each error → OpenRewrite recipe (or script); company recipes in your own module |
| Recipe run | `rewrite-maven-plugin:run` with `UpgradeSpringBoot_4_0`, on a dedicated branch |
| Prereqs | Latest 3.5.x, green build, green tests — before the recipe runs |
| Test-caught errors | Flyway never runs, hand-built mapper date format — only if tests assert schema state / serialized output |
| Silent errors | H2 console, JSpecify, null-into-primitive 400s — verify with real traffic |
| Escape hatch | `spring.jackson.use-jackson2-defaults=true` — deprecated bridge, not the end state |
| Removed | Undertow, Spock integration, classic uber-jar loader — plan around them |
| Metric | New errors per service per wave — should decay to ~zero |

Sixty services, seven errors, each solved exactly once. That's the whole playbook.
