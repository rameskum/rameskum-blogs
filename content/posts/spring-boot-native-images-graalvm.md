---
title: 'Spring Boot Native Images: Your First GraalVM Build, the Startup Numbers, and the Reflection Gotchas'
date: 2026-11-26T09:00:00-05:00
draft: false
ShowToc: true
description: >-
  Compile a Spring Boot 4.2 service to a GraalVM native image, measure the
  startup and memory win against the JVM jar, and fix the three things that
  break first — reflection, dynamic proxies, and classpath resources — with
  reachability metadata. Every number below was measured repeatedly on this
  machine; every trap was reproduced before it was fixed.
tags:
  - java
  - spring-boot
  - graalvm
  - backend
categories: article
keywords:
  - spring boot native image graalvm
  - spring boot 4.2 graalvm native image
  - graalvm native image spring boot tutorial
  - spring aot reachability metadata
  - native image reflection proxy resource
---

A Spring Boot service starts in seconds and holds a few hundred megabytes of
RAM before it serves a single request. A native image starts in milliseconds
and holds a fraction of that. The price is the closed-world assumption: the
compiler has to see everything your code can do at build time, and Spring
Boot is one of the most dynamic frameworks in the Java ecosystem.

This post builds a real service both ways and measures the difference. The
service is deliberately small — four endpoints, one of them a plain hello —
because the interesting part is not the service. It is the three things that
break the moment you leave the JVM, and the one mechanism that fixes all
three.

## The setup

Everything below ran on one machine, with the same JDK for both sides of the
comparison:

- Spring Boot **4.2.0**, Spring Framework **7.1.0**
- GraalVM CE **25.6.5.1** (JDK **25.0.5.1**), `native-image 25.0.5.1`
- GraalVM Native Build Tools Maven plugin **1.1.18** (managed by Boot 4.2.0)
- gcc 13.3 on Ubuntu 24.04 (native-image needs a C toolchain on Linux:
  `gcc`, `libc6-dev`, `zlib1g-dev` — `glibc-devel` is the Fedora name)

The JVM baseline runs on the GraalVM JDK too, so the comparison is
JIT-vs-AOT on the same runtime, not Temurin-vs-GraalVM.

One version note before the code: `spring-boot-starter-web` is deprecated in
Boot 4.2.0 in favour of `spring-boot-starter-webmvc` — the starter's own POM
says so. This post uses the current starter:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.2.0</version>
</parent>

<properties>
    <java.version>25</java.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webmvc</artifactId>
    </dependency>
</dependencies>
```

And the native plugin. Two details in this block matter, and both cost me a
failed build before I found them:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
            <configuration>
                <!-- The builder heap defaults to 80% of physical RAM.
                     Cap it, or a small machine OOMs mid-build. -->
                <jvmArgs>
                    <arg>-Xmx4g</arg>
                </jvmArgs>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

First, the `spring-boot-maven-plugin` declaration is not optional. In Boot
4.2.0 the parent puts the `repackage` execution in `pluginManagement` only,
so without this block you get a thin 5 KB jar with no `Main-Class` and no
`BOOT-INF/` — the build succeeds and the jar does nothing. Second, the
builder heap: native-image sizes its heap to 80% of physical memory by
default, which on a small VM is an OOM kill waiting to happen. `-Xmx4g`
kept this build inside a 6 GB container.

## The demo service

Four endpoints. One is the control; the other three are the traps:

```java
@SpringBootApplication
@RestController
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @GetMapping("/hello")
    public String hello() {
        return "Hello, native!";
    }

    // Trap 1: reflection — the class name is assembled at runtime, so the
    // static analysis cannot see which class will be loaded.
    @GetMapping("/reflect")
    public String reflect(@RequestParam(defaultValue = "Widget") String name) throws Exception {
        Class<?> type = Class.forName("com.example.demo." + name);
        Object widget = type.getDeclaredConstructor().newInstance();
        return "reflect:" + ((Widget) widget).name();
    }

    // Trap 2: JDK dynamic proxy — the interface is chosen at runtime, so
    // the proxy class cannot be generated at build time.
    @GetMapping("/proxy")
    public String proxy(@RequestParam(defaultValue = "Greeter") String iface) throws Exception {
        Class<?> type = Class.forName("com.example.demo." + iface);
        Object greeter = Proxy.newProxyInstance(
                DemoApplication.class.getClassLoader(),
                new Class<?>[] { type },
                (proxy, method, args) -> "Hello from a proxy!");
        return "proxy:" + ((Greeter) greeter).greet();
    }

    // Trap 3: classpath resource.
    @GetMapping("/resource")
    public String resource() throws IOException {
        try (var in = new ClassPathResource("greeting.txt").getInputStream()) {
            return "resource:" + new String(in.readAllBytes(), StandardCharsets.UTF_8).trim();
        }
    }
}
```

`Widget` is a plain POJO with a `name()` method; `Greeter` is a one-method
interface; `greeting.txt` holds one line, `hello from the classpath`. None of
the three is a Spring bean, none is referenced by any Spring-managed type —
that is the point. Spring's AOT engine can only generate hints for things it
can see.

There is a subtlety in the first two traps that the first build caught: the
class and interface names are deliberately assembled at runtime, from the
request parameter. The first version of this post used string literals —
`Class.forName("com.example.demo.Widget")` and
`new Class<?>[] { Greeter.class }` — and both worked in the native binary
with no hints at all. GraalVM's analysis resolves constant `Class.forName`
strings and literal proxy interfaces at build time and registers what they
need. The folklore that "reflection never works in native images" is wrong;
the real boundary is whether the dynamic element is a compile-time constant
or runtime data. So the traps use runtime data, the way real code does when
it loads a class from configuration or a request.

On the JVM, all four endpoints return 200, with or without parameters:

```
GET /hello                  -> Hello, native!
GET /reflect                -> reflect:widget
GET /reflect?name=Widget    -> reflect:widget
GET /proxy                  -> proxy:Hello from a proxy!
GET /proxy?iface=Greeter    -> proxy:Hello from a proxy!
GET /resource               -> resource:hello from the classpath
```

## What AOT actually does

Spring Boot's AOT (ahead-of-time) processing runs during the build, before
GraalVM compiles anything. It starts your application up to the point where
bean definitions are available — bean instances are *not* created — and
rewrites what it finds into a form the static compiler can analyse:

- **Source code generation.** `@Configuration` classes become plain factory
  code (`MyConfiguration__BeanDefinitions`) that creates bean definitions
  with direct method calls instead of reflection and proxying.
- **Hint files.** JSON under
  `META-INF/native-image/{groupId}/{artifactId}/` telling GraalVM what it
  cannot infer: `reflect-config.json`, `resource-config.json`,
  `serialization-config.json`, `proxy-config.json`, `jni-config.json`.
  Generated hints land in `target/spring-aot/main/resources`; generated
  sources in `target/spring-aot/main/sources`.
- **Proxy generation.** The proxies Spring normally creates at runtime with
  cglib are generated at build time instead.

The price is the closed-world assumption, which the Spring Boot reference
documents plainly: the beans defined in your application cannot change at
runtime. `@Profile` and profile-specific configuration have limitations, and
properties that change whether a bean is created — `@ConditionalOnProperty`
and `.enabled` properties — are not supported. If your service leans on
those, native image is a fight, not a flag.

The build itself is two commands, and the distinction between them is the
first thing that trips people up:

```bash
# AOT processing + reachability metadata only — no binary is produced.
mvn -Pnative package

# This is the command that compiles the native image.
mvn -Pnative native:compile
```

In Boot 4.2.0 the `native` profile binds `process-aot` and
`add-reachability-metadata` to the build, but it does **not** bind
`native:compile`. `mvn -Pnative package` exits `BUILD SUCCESS` in about 20
seconds with no binary anywhere — I watched it do exactly that before
reading the parent POM. The `native:compile` goal forks the lifecycle to
`package` and then runs the compiler, so the one command is enough.

## The reachability metadata repository

Before the compiler runs, the plugin pulls the GraalVM reachability
metadata repository — curated `reflect-config.json` /
`resource-config.json` / `proxy-config.json` files for common libraries,
published alongside the plugin. It is enabled by default and lives at
`org.graalvm.buildtools:graalvm-reachability-metadata:<plugin-version>:repository:zip`
on Maven Central, with a GitHub release as fallback. If your build runs
somewhere the plugin cannot reach GitHub, pre-seed that zip into your local
repository (or point the plugin at a local copy with
`<metadataRepository><localPath>…</localPath></metadataRepository>`); the
error message suggests disabling it, but that throws away the curated hints
for every library you depend on.

## The numbers

Measured the same way every time: launch the process, poll `/hello` until
the first 200, record the elapsed time, then take RSS after three warmup
requests. Four runs for the JVM jar, two each for the native binaries — the
VM has 1.5 shared CPUs, so startup times vary run to run, and the honest
number is the range, not one cherry-picked run.

JVM jar (20,369,106 bytes, `java -jar` on GraalVM 25.0.5.1):

```
run 1: launch-to-ready 6.747s, "Started in 5.288s", RSS 211,668 KiB
run 2: launch-to-ready 8.581s, "Started in 6.963s", RSS 214,596 KiB
run 3: launch-to-ready 7.842s, "Started in 5.495s", RSS 209,012 KiB
run 4: launch-to-ready 5.799s, "Started in 4.653s", RSS 209,708 KiB
```

Native binary (66,849,032 bytes, hinted build — the unhinted build produced
a byte-identical size), two runs:

```
run 1: launch-to-ready 0.133s, "Started in 0.106s", RSS 67,028 KiB
run 2: launch-to-ready 0.130s, "Started in 0.061s", RSS 66,888 KiB
```

The native binary is about 3.3× the jar on disk (66.8 MB against 20.4 MB —
the win is startup and memory, not disk), starts in about 0.13 s against the
JVM's 5.8–8.6 s, and holds about 67 MB RSS against the JVM's 209–215 MB.
The startup win is roughly 45–65×, the memory win roughly 3.2× — on a
service this small, and on a cold, CPU-starved VM. Your numbers will differ;
the direction won't.

The build itself took 13m 40s wall-clock on this machine (the unhinted build
took 13m 58s — the hints cost nothing measurable), most of it in the
native-image analysis phase, with the builder heap capped at 4 GB. That is
the trade in one sentence: you pay minutes and gigabytes at build time to
buy milliseconds and megabytes at runtime.

## Trap 1: reflection

```
GET /reflect  (native, no hints) -> HTTP 500
java.lang.ClassNotFoundException: com.example.demo.Widget
```

The failure is at `Class.forName` itself, not at the constructor call: with
the class name assembled at runtime, nothing at build time said "this class
can be loaded by name", so the class never made it into the image's runtime
class registry. The fix is a `RuntimeHintsRegistrar`, which is Spring's API
for telling AOT about dynamic behaviour it cannot see:

```java
public class NativeHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection().registerType(Widget.class,
            MemberCategory.INVOKE_DECLARED_CONSTRUCTORS);
    }
}
```

registered with `@ImportRuntimeHints(NativeHints.class)` on the application
class. After a rebuild, `/reflect` returns `reflect:widget` from the native
binary.

## Trap 2: dynamic proxies

```
GET /proxy  (native, no hints) -> HTTP 500
java.lang.ClassNotFoundException: com.example.demo.Greeter
```

The proxy itself is never reached: the endpoint fails at the same
`Class.forName` step, loading the interface by its runtime-assembled name.
A JDK dynamic proxy is a class that does not exist until runtime — the exact
thing the closed world forbids, when the interface is chosen at runtime.
(With a literal `Greeter.class`, the analysis generates the proxy at build
time and the endpoint works unhinted — same constant-vs-runtime boundary as
trap 1.) The same registrar fixes it:

```java
hints.proxies().registerJdkProxy(Greeter.class);
```

Note what AOT already handles: Spring's own proxies — `@Configuration`
enhancement, `@Transactional`, AOP advisors — are generated at build time
and work. It is the proxies *you* create with `Proxy.newProxyInstance`
that need the hint.

## Trap 3: classpath resources

```
GET /resource  (native, no hints) -> HTTP 500
java.io.FileNotFoundException: class path resource [greeting.txt]
  cannot be opened because it does not exist
```

Resources are the quietest trap of the three: nothing throws at startup,
the file simply is not there. GraalVM includes only the resources listed in
`resource-config.json`, and AOT registers the ones Spring knows about —
`application.properties`, the banner, logging config — not an arbitrary file
you load by name. Same registrar, third line:

```java
hints.resources().registerPattern("greeting.txt");
```

After the rebuild all three trap endpoints return 200 from the native
binary, alongside `/hello`.

## The complete hints class

```java
package com.example.demo;

import org.springframework.aot.hint.MemberCategory;
import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;

public class NativeHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection().registerType(Widget.class,
            MemberCategory.INVOKE_DECLARED_CONSTRUCTORS);
        hints.proxies().registerJdkProxy(Greeter.class);
        hints.resources().registerPattern("greeting.txt");
    }
}
```

```java
@SpringBootApplication
@ImportRuntimeHints(NativeHints.class)
@RestController
public class DemoApplication {
    // ...
}
```

Three lines, three traps. When the fix is bigger than the trap, you are
usually fighting the wrong layer — the common alternative, the tracing agent
(`native-image-agent`), records what your application *actually* touches at
runtime and generates the same JSON files. It is the right tool when the
dynamic behaviour is spread across libraries you do not own; the registrar
is the right tool when the behaviour is yours and you can name it.

## When it is worth it

Native image is not a faster JVM. It is a different deployment shape, and
the shape decides whether the trade pays:

- **Worth it:** short-lived processes (CLI tools, batch jobs, serverless
  functions), scale-to-zero services where cold start is the bill, and
  memory-constrained deployments where a 3× RSS cut is real money.
- **Not worth it:** long-running services where the JVM's JIT eventually
  outperforms the AOT binary, applications that lean on runtime dynamism
  (`@Profile` switching, runtime bean registration, heavy reflection),
  and any codebase where the build-time cost — minutes per build, gigabytes
  of builder RAM — lands on every CI run.
- **Check first:** run `native:list-libraries-missing-metadata` before you
  commit to the migration. It tells you which of your direct dependencies
  have no curated reachability metadata, which is the best predictor of how
  much hint-writing you are in for.

## Verification note

- Spring Boot 4.2.0 / Framework 7.1.0, GraalVM CE 25.6.5.1
  (JDK 25.0.5.1), native-maven-plugin 1.1.18. AOT behaviour and the
  closed-world restrictions verified against the Spring Boot reference
  documentation; plugin options verified against the Native Build Tools
  documentation.
- The demo service was built as a JVM jar and as a native image on the same
  machine, twice each — once without hints (all three traps reproduced with
  the exact errors above) and once with them (all endpoints 200). Startup
  and RSS measured four times for the JVM jar and twice for each native
  binary, launch-to-ready plus the `Started in …` line, with the trap
  endpoints exercised on both runtimes. The two unit tests (Widget
  behaviour) pass twice under Maven Surefire.
- Not verified: no production workload was run against either binary, and
  the numbers come from a 1.5-CPU VM — treat them as the shape of the win,
  not a benchmark. The native binary was not tested under load, and no
  container image was built from it.
