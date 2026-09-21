---
title: 'JUnit 6 Just Landed in Your Spring Boot 4.1 BOM: What Actually Changed'
date: 2026-09-21T08:50:00-04:00
draft: false
ShowToc: true
description: >-
  Spring Boot 4.1 manages JUnit 6 (6.0.3 on 4.1.0, 6.1.3 on 4.1.1) — and for
  most apps the upgrade is deleting version pins. What's new, what was
  removed, the three things that actually break (Alphanumeric ordering,
  Store.getOrComputeIfAbsent, CsvFileSource lineSeparator), plus the Surefire
  and JaCoCo bumps your build needs.
tags:
  - java
  - spring-boot
  - testing
  - junit
  - interview
categories: article
keywords:
  - junit 6
  - spring boot 4.1
  - junit 5 to 6 migration
  - testing
---

You bumped `spring-boot-starter-parent` to 4.1, ran the build, and noticed something in the test output: `junit-jupiter:6.x`. Nobody asked for it, nothing in your test code changed, and everything still passes. So what did you just get — and what breaks the day it doesn't?

Short version: JUnit 6 is mostly a cleanup release wearing a major version number. For a typical Spring Boot app, the migration is deleting lines from your `pom.xml`, not adding them. The pain lives in three specific removed APIs and two build plugins. Let's go through all of it with code.

## What the BOM actually did

Spring Boot 4.0 moved its dependency management to JUnit 6, and 4.1 kept it there. Concretely:

| Spring Boot | Managed JUnit |
|---|---|
| 4.1.0 | `junit-bom` 6.0.3 |
| 4.1.1 | `junit-jupiter` 6.1.3 |

One genuinely nice change came along for the ride: JUnit 6 uses **a single version number** for Platform, Jupiter, and Vintage. The old split — platform `1.x` vs. jupiter `5.x` — is gone. If you ever hand-wrote those versions, that's the first thing to delete.

Check what you actually resolve right now:

```bash
mvn dependency:tree -Dincludes=org.junit*
```

If you see a mix of `5.x` and `6.x` artifacts, you have a version pin or a stray BOM import fighting `spring-boot-dependencies` — which brings us to step one.

## The 5-minute migration

**1. Delete your JUnit version pins.** This is the whole migration for most projects:

```diff
 <properties>
-    <junit-jupiter.version>5.10.1</junit-jupiter.version>
 </properties>
```

And if you (or a starter you depend on) import the JUnit BOM explicitly, drop it too — `spring-boot-dependencies` already imports `junit-bom`, and a second import just produces `Ignored POM import` warnings for every managed artifact:

```diff
-        <dependency>
-            <groupId>org.junit</groupId>
-            <artifactId>junit-bom</artifactId>
-            <version>6.0.1</version>
-            <type>pom</type>
-            <scope>import</scope>
-        </dependency>
```

Let the BOM do its job. The resolved version doesn't change; the warnings disappear.

**2. Confirm your baseline is Java 17+.** JUnit 6 requires Java 17 (and Kotlin 2.2, if applicable). Spring Boot 4.x already requires 17, so if you're on Boot 4.1 you're fine. If you're somehow running tests on an older JDK, they won't just fail — the engine won't load.

**3. Upgrade Surefire/Failsafe to 3.x.** The Maven Surefire and Failsafe plugins need to be recent enough to speak to the JUnit 6 platform:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.5.2</version>
</plugin>
```

Surefire 2.x with JUnit 6 is the classic "zero tests executed, BUILD SUCCESS" trap — the build goes green because it silently ran nothing. If your test count drops to zero after the upgrade, check the plugin version before you check your tests.

**4. Keep Vintage around — temporarily.** `junit-vintage-engine` still exists for running JUnit 4 tests, but it's deprecated. Keep it while legacy tests remain, with a plan to migrate them:

```xml
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <scope>test</scope>
</dependency>
```

## What actually breaks: the three removals that bite

JUnit 6 removed every API that was deprecated in JUnit 5. Most of them you'll never touch. Three of them show up in real codebases:

**1. `MethodOrderer.Alphanumeric` is gone.** It was deprecated for years; now it's deleted.

```diff
-@TestMethodOrder(MethodOrderer.Alphanumeric.class)
+@TestMethodOrder(MethodOrderer.MethodName.class)
 class OrderServiceTest { ... }
```

`MethodName` sorts by method name (and parameter list). If you were relying on `Alphanumeric`'s exact semantics, read the Javadoc for `MethodName` — they're close but not identical.

**2. `ExtensionContext.Store.getOrComputeIfAbsent` is gone.** If you wrote custom extensions, this one will find you at compile time:

```diff
- Object instance = store.getOrComputeIfAbsent(MyKey.class, k -> new MyKey());
+ Object instance = store.computeIfAbsent(MyKey.class, k -> new MyKey());
```

The new `computeIfAbsent` overloads are also nullness-annotated (JUnit 6 adopted JSpecify annotations across all modules), so your IDE may newly flag null-handling issues in extension code. That's a feature, not a bug.

**3. `@CsvFileSource(lineSeparator = ...)` is gone.** JUnit 6 switched CSV parsing to the FastCSV library, which auto-detects line endings:

```diff
 @ParameterizedTest
-@CsvFileSource(resources = "/orders.csv", lineSeparator = "\n")
+@CsvFileSource(resources = "/orders.csv")
 void parsesOrders(String id, BigDecimal total) { ... }
```

Just delete the attribute. If your CSV files have exotic line endings that FastCSV misdetects, normalize the files — don't fight the parser.

**Honorable mention: JRE version constants.** `JRE.JAVA_8` through `JRE.JAVA_16` are deprecated, and conditions using them silently change behavior: `@EnabledOnJre(JRE.JAVA_8)` now *always skips*, `@DisabledOnJre(JRE.JAVA_8)` *never skips*. Grep for them:

```bash
grep -rn "JRE\.JAVA_" src/test --include=*.java | grep -v "JAVA_1[7-9]"
```

## The toolchain around it: JaCoCo and Mockito

**JaCoCo will break before JUnit does.** Old JaCoCo versions can't read newer class files and die with:

```
java.lang.IllegalArgumentException: Unsupported class file major version 69
    at org.jacoco.agent.rt.internal...
```

(Class file major 69 is Java 25.) If you're on JaCoCo 0.8.12 or earlier, bump it — 0.8.14 added Java 26 class-file support, and 0.8.15 covers Java 26 officially with experimental Java 27:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.15</version>
</plugin>
```

This is the single most common "JUnit 6 broke my build" report, and it isn't JUnit's fault at all.

**Mockito is fine.** Mockito 5.x works with JUnit 6 — the inline mock maker is the default since Mockito 5, so the old `mockito-inline` artifact dance is over. Just let the Boot BOM manage ByteBuddy (it does), because ByteBuddy is the piece that must understand your JDK's class files.

## What's actually new (and worth using)

Beyond the cleanup, a few additions are genuinely useful:

**Fail-fast for CI.** The ConsoleLauncher has a `--fail-fast` mode, and the platform supports cancelling execution via `CancellationToken`. For a 20-minute suite, stopping at the first failure instead of collecting all of them is a real feedback-loop win. Wire it into a profile you run on feature branches, not `main` — on main you still want the full failure list.

**Deterministic `@Nested` ordering.** JUnit 6 guarantees a deterministic execution order for `@Nested` test classes, and `@TestMethodOrder` is now inherited by nested classes. If you've ever had ordering-dependent flakiness between nested classes, this removes a whole category of it.

**JFR integration.** Flight Recorder support moved into `junit-platform-launcher` (the standalone `junit-platform-jfr` artifact is gone). You can now get JFR recordings of test execution without extra dependencies — useful when you're hunting *why* a test is slow rather than *whether* it passes.

**Single version number.** Mundane, but it kills an entire class of "platform 1.11 + jupiter 5.10 = weird behavior" dependency puzzles.

## The verdict

| Situation | Call |
|---|---|
| On Spring Boot 4.1 already | You're on JUnit 6. Delete stray pins, fix the three removals if the compiler finds them, bump JaCoCo. Done. |
| On Boot 3.x, planning the 4.x jump | JUnit 6 rides along with the upgrade. Budget an hour for the removals + plugin bumps, not a sprint. |
| Pinned JUnit 5 deliberately | Only legitimate reason is a tool in your chain that can't handle 6 yet (old IDE plugins, ancient Gradle). Check, then unpin. |
| Still on JUnit 4 via Vintage | The deprecation clock is ticking. Migrate test by test; Jupiter is largely a mechanical translation. |

The interview version of all this, if it comes up: "JUnit 6 unified the platform/jupiter/vintage versions, raised the baseline to Java 17, removed deprecated APIs like `MethodOrderer.Alphanumeric` and `Store.getOrComputeIfAbsent`, and Spring Boot 4.1 manages it via `junit-bom` — so the migration is mostly deleting version pins and bumping Surefire and JaCoCo." That's a senior-level answer in thirty seconds.

---

*References: [JUnit 6.0.0 release notes](https://docs.junit.org/6.0.0/release-notes/) · [JUnit 5→6 migration guide (OpenRewrite recipe)](https://docs.openrewrite.org/recipes/java/testing/junit6/junit5to6migration) — the recipe list doubles as a mechanical checklist if you want to automate the whole thing.*
