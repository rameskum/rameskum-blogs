---
title: 'Spring Boot 4.2 Boots Faster — Measure the Nested-JAR Fix Yourself'
date: 2026-11-10T18:00:00+05:30
draft: false
ShowToc: true
description: >-
  Spring Boot 4.2.0-RC2 is out, and its headline startup change is invisible
  in your code: classes and resources are now looked up directly in the
  nested jars of an executable jar instead of opening a URL connection per
  lookup. I measured it — same app, same JVM, five runs each — and checked
  the two other changes that actually bite: the Ubuntu Resolute buildpack
  image and the properties migrator that can now fail your build.
tags:
  - java
  - spring-boot
  - backend
categories: article
keywords:
  - spring boot 4.2 startup time
  - spring boot 4.2.0-RC2 release notes
  - nested jar class lookup
  - spring boot properties migrator fail
  - paketo ubuntu resolute builder
---

Spring Boot 4.2.0-RC2 landed on November 6 with 99 enhancements, and GA is planned for November 30. If you're still on 4.0, the clock matters too: its free support ends December 31, 2026. But the reason to read the release notes isn't the version number — it's one line in the RC1 notes under "Startup Performance Improvements":

> Classes and resources are now looked up directly in the nested jars of an executable jar, instead of opening a URL connection for every nested jar on every lookup.

No code change, no flag, no new annotation. You upgrade the version and the lookup gets cheaper. The question is how much cheaper — so I measured it instead of quoting it.

## What actually changed inside the jar

An executable Boot jar is a jar of jars: your classes in `BOOT-INF/classes/`, your dependencies as nested jars in `BOOT-INF/lib/`, and the loader (`JarLauncher`) that builds a classloader over them. Every time the JVM needs a class or a resource, the loader has to find which nested jar holds it. Before 4.2, that lookup went through a `URLConnection` for every nested jar on every lookup — open the connection, navigate to the entry, close it. With 50 dependencies, that's 50 connection opens per lookup, thousands of lookups per startup, and a lot of allocation along the way. In 4.2 the loader reads the nested jars' indexes directly.

The same release note adds three more startup items: reading deflated nested jar entries is optimized, resolving cached resources is optimized, and sorting auto-configuration classes and DevTools restarts are faster. The auto-configuration sorting one is easy to miss and easy to feel — on a big app with hundreds of auto-configurations, the sort runs on every startup.

One thing the change does *not* do: it doesn't change your code, your jar layout, or your launcher. The improvement is entirely inside `spring-boot-loader`. That also means the measurement below is a pure version comparison — same app, same dependencies, same JVM, only the Boot version moves.

## The measurement

The app is deliberately boring: `spring-boot-starter-web` + `spring-boot-starter-actuator`, one `@RestController` with a single endpoint, random port, no database. The interesting part is the harness: the app records the time in `main()` and prints the elapsed milliseconds when the `ApplicationReadyEvent` fires, then exits — so each run is a cold JVM start, and the number is startup time, not steady-state anything.

```java
@SpringBootApplication
@RestController
public class StartupApp {

    private static final long START_NANOS = System.nanoTime();

    public static void main(String[] args) {
        SpringApplication.run(StartupApp.class, args);
    }

    @GetMapping("/hello")
    public String hello() {
        return "hello";
    }

    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        long elapsedMs = (System.nanoTime() - START_NANOS) / 1_000_000;
        System.out.println("STARTUP_MS=" + elapsedMs);
        System.out.flush();
        System.exit(0);
    }
}
```

Both versions were built as real executable jars (loader + `BOOT-INF/lib`, the way the Maven plugin builds them — more on the hand-rolled layout in a moment) and run five times each, alternating versions to cancel out machine drift, on Temurin JDK 25. Two numbers per run: `STARTUP_MS` (class-load to ready) and wall time (process spawn to exit).

| Run | 4.1.2 STARTUP_MS | 4.2.0-RC2 STARTUP_MS | 4.1.2 wall | 4.2.0-RC2 wall |
| --- | --- | --- | --- | --- |
| 1 | 6998 | 6502 | 8135 | 6917 |
| 2 | 6485 | 6000 | 6441 | 5979 |
| 3 | 6659 | 5279 | 7064 | 6180 |
| 4 | 5932 | 5494 | 6578 | 6187 |
| 5 | 6319 | 5764 | 6234 | 5708 |
| **Median** | **6485** | **5764** | **6578** | **6180** |
| **Mean** | 6479 | 5808 | 6890 | 6194 |

Median startup: 6485ms → 5764ms, about **721ms (11%) faster**. Median wall time: 6578ms → 6180ms, about **398ms (6%) faster**. The means agree: roughly 10% faster on both measures.

Read the numbers honestly, though. The run-to-run variance is wide — 4.1.2's fastest run (5932ms) beats 4.2.0-RC2's slowest (6502ms), and this is a small shared-CPU VM where a noisy neighbor moves the numbers by hundreds of milliseconds. The median gap is consistent across both measures and five alternating runs, so the improvement is real, but the honest statement is "about 10% on a small app on a small machine", not a precise figure. Two things would make it bigger: more nested jars (the fix scales with lookup count, so a 150-dependency monolith gains more than my 50-jar app) and a faster disk (the old path paid per-lookup connection overhead that a slow filesystem amplifies). And the release notes claim lower allocation during startup too — I didn't measure that, so I won't put a number on it.

If you want to reproduce this on your own app, the method is the whole trick: same jar layout, same JVM flags, alternate the versions run by run, and report the median, not the best run. The best run is a lottery ticket.

## The hand-rolled jar lesson: nested jars must be stored, not deflated

Building the fat jars by hand taught me something the Maven plugin hides: **nested jars in `BOOT-INF/lib/` must be stored uncompressed**. My first build compressed them (the default for most zip tools) and the app died immediately:

```
Caused by: java.lang.NoClassDefFoundError: org/springframework/boot/SpringApplication
Caused by: java.lang.ClassNotFoundException: org.springframework.boot.SpringApplication
```

The loader could see the nested jars — its debug log listed every one — but it couldn't load classes out of them. Rebuilding with the nested jars stored (`ZIP_STORED`) fixed it, and the same failure reproduced on a clean dependency set in both 4.1.2 and 4.2.0-RC2. The loader memory-maps nested jars, and a deflated nested jar can't be mapped. The `spring-boot-maven-plugin` and the Gradle plugin store them uncompressed by default, which is why you've never hit this — but if you ever hand-roll a Boot jar, or write a tool that repacks one, this is the failure mode, and the error message gives you no hint that compression is the cause.

Note what this means for the 4.2 note about "reading deflated nested jar entries" being optimized: that refers to deflated *entries inside* the nested jars (your class files are deflated inside their jars, as usual), not to deflated nested jars themselves. I verified the distinction directly — a deflated-nested-jar build fails identically on the 4.2.0-RC2 loader.

## Change two: the buildpack image moves to Ubuntu Resolute

The RC1 notes change the default builder for the Maven and Gradle plugins from `paketobuildpacks/builder-noble-java-tiny` to `paketobuildpacks/ubuntu-resolute-builder`. Your images are now based on Ubuntu 26.04 instead of 24.04, and the default run image is `paketobuildpacks/ubuntu-resolute-run-tiny:latest`.

Two things in that sentence deserve attention. First, the run image is **no longer pinned to a specific version by the builder** — `:latest` moves under you, so a rebuild next month can silently change the base OS. If you need reproducible images, pin both `builder` and `runImage` to specific versions. Second, the plugins now check run-image compatibility by OS distribution labels instead of the deprecated stack IDs, and log a warning when they differ — so mixing the new Resolute builder with an old Noble run image will tell you about it instead of failing obscurely. To keep the old builder, set `builder` to `paketobuildpacks/builder-noble-java-tiny`.

I didn't run a buildpack build for this post (no Docker daemon on the measurement VM), so the image names and the pinning behavior are verified against the release notes, not a live build — the startup and migrator sections below are the parts I ran.

## Change three: the properties migrator can now fail your build

The `spring-boot-properties-migrator` has always been the "turn it on, read the warnings, turn it off" tool: add the dependency, boot once, and it logs every property you've set that was renamed or removed, along with its replacement. In 4.2 it gains a fail switch, `spring.properties.migrator.fail`:

```properties
spring.properties.migrator.fail=never       # default: log warnings, start anyway
spring.properties.migrator.fail=on-error    # fail on properties that no longer exist
spring.properties.migrator.fail=on-warning  # fail on renamed or removed properties
```

I verified both ends with a real renamed property. The app's `application.properties` sets `spring.resources.static-locations`, which was renamed to `spring.web.resources.static-locations` back in Boot 2.4, plus the migrator on the classpath. With the default, the app starts and logs:

```
WARN  ... o.s.b.c.p.m.PropertiesMigrationListener  :
	Key: spring.resources.static-locations
		Replacement: spring.web.resources.static-locations
```

With `spring.properties.migrator.fail=on-warning`, the same app refuses to start:

```
ERROR ... o.s.boot.SpringApplication : Application run failed

java.lang.IllegalStateException: Found configuration keys that need to be migrated
```

That is the right shape for a CI gate: add the migrator with `on-warning` in a test profile, and a renamed property fails the build instead of silently running with a dead setting. Two practical notes from the run: the migrator needs `spring-boot-configuration-metadata` on the classpath — a separate artifact, and without it the migrator fails with a `NoClassDefFoundError` that looks nothing like a migration error — and the docs' advice still stands — remove the migrator dependency once the migration is done; it's a diagnostic tool, not a runtime crutch.

## The rest of RC2, in brief

These are verified against the RC1/RC2 release notes; I didn't exercise them live:

- **Redis database on master/replica.** `spring.data.redis.database` (or the database in `spring.data.redis.url`) was silently ignored with `spring.data.redis.masterreplica.nodes` — the factory always used database 0. In 4.2 the configured database is applied. If you were unknowingly relying on database 0, set it to `0` explicitly.
- **X-Forwarded-Prefix ignored by default (WebFlux).** With `server.forward-headers-strategy=native`, Reactor Netty no longer uses `X-Forwarded-Prefix` to determine the context path; same for the `framework` strategy with WebFlux. Behind a proxy that sets the header, opt back in with `server.netty.forwarded-headers.use-forwarded-prefix` or `spring.webflux.forwarded-headers.use-forwarded-prefix`.
- **Unsupported SSL bundle options fail startup.** SSL bundle `options.ciphers` and `options.enabled-protocols` were silently ignored by gRPC, the LDAP client, the embedded LDAP server, and the OTLP exporters. They now fail startup — consistent with MongoDB and Couchbase. Also note: SSL bundle reloading is no longer supported by the LDAP client at all.
- **OTLP semantic conventions.** Setting `management.observations.conventions` to `open-telemetry` now actually enables OpenTelemetry's semantic conventions; previously neither Micrometer's nor OpenTelemetry's were applied.

## Should you upgrade?

The timeline: RC2 is out now, GA is planned for November 30, 2026, and Boot 4.0's free support ends December 31, 2026. If you're on 4.1, this is a minor upgrade — the breaking changes above are the short list, and the migrator with `on-warning` is the cheapest way to find out if any of them touch you. If you're on 4.0, the 4.2 GA date and the 4.0 EOL date are a month apart; plan the 4.1 hop first, then 4.2, rather than jumping both at once. And if you're on 3.x, none of this is your next step — the 4.0 migration guide is.

The startup gain alone doesn't justify an upgrade on a small service — 700ms on a 6.5-second boot is nice, not decisive. What justifies it is the combination: a faster boot, a buildpack base image that won't surprise you if you pin it now, and a migrator that can finally enforce the migration instead of whispering about it. Upgrade in a branch, boot it with the migrator on `on-warning`, and let the app tell you what else needs attention.

## Sources

- Spring Boot 4.2.0-RC1 release notes (GitHub wiki) — startup improvements, Ubuntu Resolute builder, properties migrator fail support.
- Spring Boot 4.2.0-RC2 release notes (GitHub wiki) — Redis database on master/replica, X-Forwarded-Prefix, SSL bundle options, RabbitMQ AMQP 1.0 refinements, OTLP conventions.
- Spring Boot 4.2.0-RC2 announcement (spring.io blog, Nov 6, 2026) — 99 enhancements, available on Maven Central.
- Spring Boot upgrading docs, 4.2-SNAPSHOT (docs.spring.io) — `spring.properties.migrator.fail`: `never` / `on-error` / `on-warning`.
- Spring project generations API (api.spring.io) — 4.2.x initial 2026-11-30, OSS ends 2027-12-31; 4.0.x OSS ends 2026-12-31; 4.1.x OSS ends 2027-07-31.
- Measured on this VM: same app (spring-boot-starter-web + actuator, one endpoint) as executable jars on Temurin JDK 25.0.5, 5 alternating runs per version — STARTUP_MS median 6485ms (4.1.2, Framework 7.0.10) vs 5764ms (4.2.0-RC2, Framework 7.1.0-RC2); wall-time median 6578ms vs 6180ms. Migrator fail/warn behavior and the stored-vs-deflated nested jar failure both reproduced live on both versions. No Docker daemon on this VM, so the buildpack image change was verified against the release notes, not a live build.
