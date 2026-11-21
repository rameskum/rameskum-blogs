---
title: 'Build Your First MCP App with Spring AI 2.1: A Weather Server, a Tool-Calling Client, and the Session API That Replaces ChatMemory'
date: 2026-11-21T08:31:00-05:00
draft: false
ShowToc: true
description: >-
  MCP is the USB-C port for AI apps: one protocol for tools, resources, and
  prompts, and every model client speaks it. Spring AI 2.1 (GA November 20,
  2026) makes a Spring Boot app both sides of that port — a server in two
  annotations, a client in one property block — and replaces the deprecated
  ChatMemory with a versioned Session event log. This post builds the whole
  loop with no API key and no real LLM: a weather MCP server, a ChatClient
  that calls its tool, and session memory that remembers across turns. Seven
  tests, run three times, every number below measured.
tags:
  - spring-boot
  - spring-ai
  - java
  - mcp
  - ai
categories: article
keywords:
  - spring ai mcp server
  - spring ai 2.1 mcp
  - model context protocol spring boot
  - spring ai session api
  - chatmemory deprecated
  - mcp streamable http modern protocol
---

Your LLM knows a lot, but it does not know the weather in Toronto, the balance in your database, or the status of your build. The Model Context Protocol fixes that with one idea: the model asks for a tool by name, something on the other end runs it, and the result comes back as a message. The protocol is the same whether the tool is local or across the network, whether the client is Claude Desktop or your own Spring Boot service. It is the USB-C port for AI apps.

Spring AI 2.1, released November 20, 2026, is the first version that treats MCP as a first-class citizen instead of a side project. It ships on Spring Boot 4.2, it speaks the MCP 2026-07-28 specification through the MCP Java SDK 2.2.0, and it quietly retires the `ChatMemory` API you have been using since Spring AI 1.0 in favour of a new Session API. This post builds the whole loop, end to end, with no API key and no real language model: a weather MCP server, a `ChatClient` that calls its tool, and a session that remembers the conversation across turns. A stub model stands in for the LLM — it "decides" to call the tool on the first turn and answers from the result on the second — so every behaviour below is measured, not assumed. Swap the stub for an OpenAI or Ollama `ChatModel` bean and the wiring does not change.

## The stack

| Piece | Version | Notes |
| --- | --- | --- |
| Spring Boot | 4.2.0 | parent POM, Java 17 baseline |
| Spring Framework | 7.1.0 | Boot 4.2's version |
| Spring AI | 2.1.0 | GA November 20, 2026; MCP Java SDK 2.2.0 underneath |
| Jackson | 3.1.8 | Boot 4.2's stack (`tools.jackson`) |
| JDK | 25.0.5.1 | Temurin, the Java4Shell fix build |
| JUnit | 6.1.3 | tests run three times, 7/7 every time |

The dependencies are the two MCP starters, the session advisor, and the auto-configured `ChatClient.Builder`:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.2.0</version>
</parent>

<properties>
    <spring-ai.version>2.1.0</spring-ai.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- the MCP server (webmvc transport) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
    </dependency>
    <!-- the MCP client (stdio + streamable-http) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-mcp-client-webflux</artifactId>
    </dependency>
    <!-- the new Session API: memory advisor + auto-configuration -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-session-advisor</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-autoconfigure-model-session</artifactId>
    </dependency>
    <!-- the auto-configured ChatClient.Builder bean -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-autoconfigure-model-chat-client</artifactId>
    </dependency>
</dependencies>
```

Two things worth noticing before any code runs. First, the client starter is WebFlux-based even when the server is WebMVC — the client and server transports are independent choices, and the stdio client does not need a web stack at all. Second, `spring-ai-autoconfigure-model-chat-client` is a separate artifact: without it there is no `ChatClient.Builder` bean, and the documented `ChatClient` path does not work. The programmatic `ChatClient.builder(model)` still exists, but the Spring AI docs are explicit that programmatic creation bypasses auto-configuration, and `ChatClientBuilderConfigurer` is the supported programmatic route.

## Part 1: the server is two annotations

The whole server is one component and one `main` method:

```java
package mcp.server;

import org.springframework.ai.mcp.annotation.McpResource;
import org.springframework.ai.mcp.annotation.McpTool;
import org.springframework.ai.mcp.annotation.McpToolParam;
import org.springframework.stereotype.Component;

@Component
public class WeatherTools {

    @McpTool(name = "get_forecast", description = "Get the weather forecast for a city")
    public String getForecast(@McpToolParam(description = "City name, e.g. Toronto") String city) {
        return "Sunny, 21°C in " + city + ".";
    }

    @McpResource(uri = "weather://cities", name = "cities",
            description = "Cities with a forecast available")
    public String cities() {
        return "Toronto, Montreal, Vancouver";
    }
}
```

```java
@SpringBootApplication
public class WeatherServerApp {
    public static void main(String[] args) {
        SpringApplication.run(WeatherServerApp.class, args);
    }
}
```

`@McpTool` turns a method into a tool; `@McpToolParam` describes one argument; `@McpResource` exposes a readable URI. The annotations live in `org.springframework.ai.mcp.annotation`. There is no controller, no JSON-RPC handler, no endpoint class to write — the `spring-ai-starter-mcp-server-webmvc` auto-configuration registers the transport, the tool registry, and the resource registry. The server's identity comes from two properties:

```properties
spring.ai.mcp.server.name=weather-server
spring.ai.mcp.server.protocol=STREAMABLE
```

`serverInfo.name` in every handshake below reads `weather-server` back, which is how you confirm the property landed. The default version is `1.0.0`.

### Trap 1: `@McpToolParam` has no name — the bytecode does

Look at the parameter annotation again: it has `required`, `description`, and `header`, but no `name`. The tool's JSON schema takes its parameter names from the bytecode, which means it takes them from the `-parameters` compiler flag. Compile without it and the schema says `arg0`, and every call fails server-side validation:

```json
{"isError": true, "content": [{"type": "text",
  "text": "required property 'arg0' not found"}]}
```

That is the error you get when the client sends `{"city": "Toronto"}` and the server's schema expects `arg0`. If you build with Maven and Boot's parent POM you never see this — the parent sets `<parameters>true</parameters>` in the compiler plugin — but a raw `javac` invocation, a Gradle build without the flag, or a hand-rolled build script will silently produce `arg0` tools. Compile with `javac -parameters` (or check your build enables it) and the schema says `city`, as every transcript below shows.

### Trap 2: a stdio server must log to stderr

The same server runs over stdio as well as HTTP — `--spring.ai.mcp.server.stdio=true --spring.main.web-application-type=none` — and stdio means the JSON-RPC stream owns stdout. Any log line on stdout corrupts the stream. Route everything to stderr:

```xml
<!-- src/test/resources/logback.xml -->
<configuration>
    <appender name="STDERR" class="ch.qos.logback.core.ConsoleAppender">
        <target>System.err</target>
        <encoder><pattern>%d{HH:mm:ss} %-5level %logger{36} - %msg%n</pattern></encoder>
    </appender>
    <root level="WARN"><appender-ref ref="STDERR"/></root>
</configuration>
```

This is the kind of thing that works in every tutorial and breaks in your first real run, because the default logback configuration writes to stdout and the first log line kills the handshake with a JSON parse error on the client side.

### The stdio handshake, measured

The real MCP Java SDK sync client talking to the real server as a subprocess — `initialize()`, `listTools()`, `callTool`, `listResources()`:

```java
var params = ServerParameters.builder(java)
        .args("-cp", classpath, "mcp.server.WeatherServerApp",
                "--spring.ai.mcp.server.stdio=true",
                "--spring.main.web-application-type=none",
                "--spring.main.banner-mode=off")
        .build();
var transport = new StdioClientTransport(params,
        new JacksonMcpJsonMapper(JsonMapper.builder().build()));
try (var client = McpClient.sync(transport)
        .requestTimeout(Duration.ofSeconds(30))
        .build()) {
    McpSchema.InitializeResult init = client.initialize();
    var tools = client.listTools().tools();
    McpSchema.CallToolResult result = client.callTool(
            new McpSchema.CallToolRequest("get_forecast",
                    Map.of("city", "Toronto"), null));
}
```

Measured: `initialize()` negotiates protocol version `2025-11-25`; `listTools()` contains `get_forecast`; `callTool` returns exactly `Sunny, 21°C in Toronto.` with `isError=false`; `listResources()` contains `weather://cities`. The three-argument `CallToolRequest(name, args, meta)` is the current form — the two-argument constructor is deprecated for removal in 2.1.0.

## Part 2: four protocols, one endpoint

`spring.ai.mcp.server.protocol` takes four values: `SSE`, `STREAMABLE`, `STATELESS`, `MODERN`. The default is `STREAMABLE` (verified in the `McpServerProperties` bytecode — the field initialises to `STREAMABLE`). The endpoint is `/mcp` in every case. Here is what each one actually does on the wire, measured with `curl` against the same server on port 18080.

### STREAMABLE (the default): sessions and SSE

```http
POST /mcp
Content-Type: application/json
Accept: application/json, text/event-stream

{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
  "protocolVersion":"2025-11-25","capabilities":{},
  "clientInfo":{"name":"curl","version":"1.0"}}}
```

```http
HTTP/1.1 200
Mcp-Session-Id: 68e7fbdd-2992-4480-97c1-8051ee1e8abf
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"result":{
  "protocolVersion":"2025-11-25",
  "capabilities":{"completions":{},"logging":{},
    "prompts":{"listChanged":true},
    "resources":{"subscribe":false,"listChanged":true},
    "tools":{"listChanged":true}},
  "serverInfo":{"name":"weather-server","version":"1.0.0"}}}
```

The server returns a session id in the `Mcp-Session-Id` header, and every later call carries it back. `notifications/initialized` answers `202`. `tools/list` and `tools/call` come back as server-sent events:

```
id:68e7fbdd-2992-4480-97c1-8051ee1e8abf
event:message
data: {"jsonrpc":"2.0","id":3,"result":{
  "content":[{"type":"text","text":"Sunny, 21°C in Toronto."}],
  "isError":false}}
```

`tools/list` shows `get_forecast` with the `city` string property, required, and the description from `@McpToolParam`. This is the protocol most clients speak today: a JSON-RPC handshake, a session, and SSE for the responses.

### STATELESS: same calls, no session, plain JSON

Set `spring.ai.mcp.server.protocol=STATELESS` and the handshake changes in two ways: there is no `Mcp-Session-Id` header at all, and the responses are plain JSON instead of SSE. Everything else is identical — same `serverInfo`, same `tools/list`, same `tools/call` result. Stateless is the protocol for a server behind a load balancer that cannot pin a client to one instance, or for a server that keeps no per-client state: each request is self-contained. The one visible difference besides the session header is the capabilities block — the stateless handshake omits `logging`, because there is no session to push log notifications over.

### MODERN: no initialize, discover, and cacheable results

`MODERN` is the MCP 2026-07-28 specification, and it drops the handshake entirely. There is no `initialize` and no session. The client starts with `server/discover`, and the wire format changes shape: the method travels in an `Mcp-Method` header, the protocol version in an `MCP-Protocol-Version` header, and every request carries a `params._meta` block with the protocol version, the client capabilities, and the client info:

```http
POST /mcp
Mcp-Method: server/discover
MCP-Protocol-Version: 2026-07-28
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"server/discover","params":{
  "_meta":{
    "io.modelcontextprotocol/protocolVersion":"2026-07-28",
    "io.modelcontextprotocol/clientCapabilities":{},
    "io.modelcontextprotocol/clientInfo":{"name":"curl","version":"1.0"}}}}
```

```http
HTTP/1.1 200
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"result":{
  "supportedVersions":["2026-07-28"],
  "capabilities":{"completions":{},"prompts":{"listChanged":true},
    "resources":{"subscribe":false,"listChanged":true},
    "tools":{"listChanged":true}},
  "ttlMs":0,"cacheScope":"private","resultType":"complete",
  "_meta":{"io.modelcontextprotocol/serverInfo":
    {"name":"weather-server","version":"1.0.0"}}}}
```

Two things in that response matter. First, `supportedVersions` is a list — the client picks a version instead of negotiating one through `initialize`. Second, `ttlMs` and `cacheScope` are the point of the whole redesign: discovery and list results are cacheable, so a client (or a CDN in front of the server) can serve them without hitting the server again. `tools/call` works the same way, with one more header — `Mcp-Name: get_forecast` carries the tool name alongside `Mcp-Method: tools/call` — and the result carries the same `resultType: "complete"` and `serverInfo` in `_meta`:

```json
{"jsonrpc":"2.0","id":3,"result":{
  "content":[{"type":"text","text":"Sunny, 21°C in Toronto."}],
  "isError":false,"resultType":"complete",
  "_meta":{"io.modelcontextprotocol/serverInfo":
    {"name":"weather-server","version":"1.0.0"}}}}
```

The modern Java client mirrors this shape — `McpClient.sync(HttpClientMcpTransport.builder(url).build()).build()` gives you `discover()` (with `.supportedVersions()`), `listTools()`, and `tool(name).arguments(Map).call()` — and one hard rule: MODERN is sync-only. Configuring `ASYNC` with `MODERN` fails at startup. `SSE` is the fourth value, the legacy 2025-03-26 transport; it still exists, but nothing in this post needs it, and STREAMABLE is the default for a reason.

## Part 3: the client — one property block, one provider bean

The client side is a stdio connection in properties and a `ToolCallbackProvider` bean in the `ChatClient`:

```properties
spring.ai.mcp.client.stdio.connections.weather.command=java
spring.ai.mcp.client.stdio.connections.weather.args=-cp,<classpath>,mcp.server.WeatherServerApp,--spring.ai.mcp.server.stdio=true,--spring.main.web-application-type=none,--spring.main.banner-mode=off
```

The `args` value is comma-separated and binds to a list. For an HTTP server the equivalent is `spring.ai.mcp.client.streamable-http.connections.<name>.url=http://localhost:18080/mcp`. The auto-configuration (`McpToolCallbackAutoConfiguration`) then creates a bean named `mcpToolCallbacks` — a `SyncMcpToolCallbackProvider` — as long as `spring.ai.mcp.client.enabled` and `spring.ai.mcp.client.toolcallback.enabled` are on (both default to true) and the client type is `SYNC` (also the default). The `ChatClient` takes it through `defaultTools`:

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder,
        @Qualifier("mcpToolCallbacks") ToolCallbackProvider toolCallbackProvider) {
    return builder.defaultTools(toolCallbackProvider).build();
}
```

Two deprecations to know about, both for removal: every `defaultToolCallbacks(...)` overload is deprecated — use `defaultTools(...)` — and the `SyncMcpToolCallbackProvider` constructors are deprecated in favour of `builder()` with `.mcpClients(...)` and `.toolNamePrefixGenerator(...)`. The default prefix generator does not prefix anything: the callback name is the plain `get_forecast`, which is what the model sees.

### The stub model, and why it is honest

No real LLM is called anywhere in this post. The model is a stub that answers from a script, so the plumbing can be tested without an API key:

```java
public class StubChatModel implements ChatModel {
    private final Function<Prompt, ChatResponse> script;
    final List<Prompt> seenPrompts = new ArrayList<>();

    @Override
    public ChatResponse call(Prompt prompt) {
        seenPrompts.add(prompt);
        return script.apply(prompt);
    }
    // ...
}
```

The test's script returns a tool call on the first invocation and a final answer on the second:

```java
var calls = new AtomicInteger();
StubChatModel stub = new StubChatModel(prompt -> {
    if (calls.incrementAndGet() == 1) {
        return StubChatModel.toolCall("call-1", "get_forecast",
                "{\"city\": \"Toronto\"}");
    }
    return StubChatModel.text("The forecast for Toronto is sunny, 21°C.");
});
```

The point of recording every prompt is that the test can assert what the model actually saw — not just the final answer. And the final answer is the proof of the whole loop: the test asserts the answer contains `sunny, 21°C`, the model was called exactly twice, and the second prompt contains the tool's real return value, `Sunny, 21°C in Toronto.`, which only exists if the MCP server ran the tool. That is the test:

```java
@Test
void chatClientCallsMcpToolAndAnswersFromResult() {
    String answer = chat.prompt("What is the weather in Toronto?")
            .call().content();
    assertTrue(answer.contains("sunny, 21°C"), "answer was: " + answer);
    assertEquals(2, stub.seenPrompts.size(),
            "model called twice: tool call, then final answer");
    assertTrue(stub.seenPrompts.get(1).getInstructions().toString()
                    .contains("Sunny, 21°C in Toronto."),
            "the tool result must reach the model on the second call");
}
```

It passes — but it took two traps to get there, and both are worth more than the passing test.

### Trap 3: the ToolCall record's argument order

The stub builds the tool call like this:

```java
static ChatResponse toolCall(String id, String name, String argumentsJson) {
    var msg = AssistantMessage.builder()
            .toolCalls(List.of(new AssistantMessage.ToolCall(
                    id, "function", name, argumentsJson)))
            .build();
    return new ChatResponse(List.of(new Generation(msg)));
}
```

The record's components are `(id, type, name, arguments)` — not `(id, name, type, arguments)`. Pass them in the wrong order and the tool call's *name* becomes `"function"` and its *type* becomes `"get_forecast"`. The advisor then looks for a callback named `function`, finds none, and the tool never executes. The symptom is an empty answer: the tool-call response has no text, so `.content()` — which reads the text of the final generation — returns `""`. Nothing throws. Nothing logs an error. The loop just silently returns the tool-call message as if it were the answer. If your tool loop ever returns an empty string, check the `ToolCall` constructor first.

### Trap 4: the model declares tool support, not the client

This one cost the most time. With the argument order fixed, the test still returned an empty answer and the tool still never ran. The cause was in `DefaultChatClientUtils`: tool callbacks are only wired into the prompt when the model's options mutate into a `ToolCallingChatOptions.Builder`:

```java
ChatOptions.Builder<?> builder = inputRequest.getChatModel().getOptions().mutate();
if (builder instanceof ToolCallingChatOptions.Builder<?> tbuilder) {
    // ... attach the tool callbacks to the options
}
```

And the `ToolCallingAdvisor` skips itself entirely when the prompt's options are not `ToolCallingChatOptions`:

```java
ChatOptions options = chatClientRequest.prompt().getOptions();
if (!(options instanceof ToolCallingChatOptions toolCallingChatOptions)) {
    // No tool calling options - skip the tool calling advisor
    return callAdvisorChain.nextCall(chatClientRequest);
}
```

My stub inherited the default `getOptions()` from `ChatModel`, which returns plain `ChatOptions`. So the advisor was in the chain, the tools were configured, the model returned a tool call — and the advisor walked straight past it. The fix is one method on the stub:

```java
// The tool loop only runs when the model's options are ToolCallingChatOptions.
// A plain ChatOptions here silently skips the ToolCallingAdvisor.
@Override
public ChatOptions getOptions() {
    return ToolCallingChatOptions.builder().build();
}
```

Real models already do this — OpenAI's and Ollama's options extend `ToolCallingChatOptions` — which is why the trap only bites stubs and custom `ChatModel` implementations. But it bites silently, and the symptom (empty answer, no error) points nowhere near the cause. The lesson generalises: in Spring AI 2.1, tool calling is a property of the *model*, not the client. The client can offer tools all day; if the model does not declare that it supports them, nothing runs.

## Part 4: the Session API replaces ChatMemory

The second half of Spring AI 2.1 is the Session API, and it comes with a deprecation that matters: the whole `ChatMemory` family is deprecated in 2.1.0 and slated for removal in 3.0.0. The exception is `VectorStoreChatMemoryAdvisor`, which is *not* deprecated — the retrieval-backed memory advisor stays. Everything else moves to `org.springframework.ai.session`.

The model is an append-only event log, not a message list. A `SessionService` holds sessions; every write is an event with a version; reads distinguish the full log from the active view:

```java
sessions = DefaultSessionService.builder()
        .sessionRepository(InMemorySessionRepository.builder().build())
        .build();
chat = ChatClient.builder(stub)
        .defaultAdvisors(SessionMemoryAdvisor.builder(sessions).build())
        .build();
```

The advisor is wired the same way the old `MessageChatMemoryAdvisor` was, and the conversation id travels the same way — `MemoryAdvisor.CONVERSATION_ID` (`"chat_memory_conversation_id"`) as an advisor parameter:

```java
chat.prompt("My name is Ada")
        .advisors(a -> a.param(MemoryAdvisor.CONVERSATION_ID, "conv-1"))
        .call().content();
chat.prompt("What is my name?")
        .advisors(a -> a.param(MemoryAdvisor.CONVERSATION_ID, "conv-1"))
        .call().content();
```

Measured: the second prompt contains the first turn's user message (`My name is Ada`), and `sessions.getMessages("conv-1")` holds 4 messages — two user, two assistant — after two turns. The advisor replays the session into the prompt before the model call, exactly as the old memory advisor did. The difference is what sits underneath.

### Version-checked appends

The event log is versioned, and appends are conditional on the version you read:

```java
sessions.create(CreateSessionRequest.builder().id("versioned").build());
long v0 = sessions.getEventVersion("versioned");
sessions.appendMessage("versioned", new UserMessage("hello"));
long v1 = sessions.getEventVersion("versioned");
assertEquals(v0 + 1, v1);

var fresh = SessionEvent.builder()
        .id(UUID.randomUUID().toString())
        .sessionId("versioned")
        .type(EventType.MESSAGE)
        .message(new UserMessage("second"))
        .build();
assertFalse(sessions.appendEvents("versioned", List.of(fresh), v0),
        "stale expectedVersion must be rejected");
assertEquals(v1, sessions.getEventVersion("versioned"),
        "rejected append must not move the version");
assertTrue(sessions.appendEvents("versioned", List.of(fresh), v1),
        "current expectedVersion must be accepted");
```

A stale `expectedVersion` is rejected and the version does not move; the current version is accepted and the version increments by one. This is optimistic concurrency for conversations: two writers racing on the same session cannot silently interleave their turns. One sharp edge, found by running the test: re-appending the *same* events throws `Event id ... is already in the log` — event ids are unique per session, so build fresh events for each append (note the `UUID.randomUUID()` above).

### Compaction keeps whole turns

The old API truncated messages by count, which could cut a turn in half — a user message without its answer, or an answer without its question. The Session API compacts by *turns*:

```java
// 4 turns = 8 messages
for (int i = 1; i <= 4; i++) {
    sessions.appendMessage("compact-me", new UserMessage("question " + i));
    sessions.appendMessage("compact-me", new AssistantMessage("answer " + i));
}
sessions.compact("compact-me", new TurnCountTrigger(2),
        WindowCompactionStrategy.builder()
                .window(CompactionWindow.lastTurns(1)).build());

var active = sessions.getActiveMessages("compact-me");
assertEquals(2, active.size(), "only the last turn stays active");
assertTrue(active.get(0).getText().contains("question 4"));
assertTrue(active.get(1).getText().contains("answer 4"));
assertEquals(8, sessions.getMessages("compact-me").size(),
        "the full log is kept; only the view shrinks");
```

Measured: after compaction the *active* view holds exactly the last turn (question 4 and answer 4 — a whole turn, never half of one), while the full log still holds all 8 messages. Compaction changes what the advisor replays into the prompt; it does not delete history. The trigger (`TurnCountTrigger(2)`) decides *when* to compact; the strategy (`lastTurns(1)`) decides *what* stays active.

## What was verified, and what was not

Seven tests, run three times on Spring AI 2.1.0, Spring Boot 4.2.0, Framework 7.1.0, and Temurin JDK 25.0.5.1 — 7/7 every time. The stdio server handshake (`initialize` negotiating `2025-11-25`, `listTools`, `callTool` returning `Sunny, 21°C in Toronto.`, `listResources`) runs against the real server subprocess. The HTTP transports run against the real webmvc server with `curl`: STREAMABLE with its session header and SSE responses, STATELESS with plain JSON and no session, MODERN with `server/discover`, the `_meta` block, the `Mcp-Method` / `Mcp-Name` / `MCP-Protocol-Version` headers, and cacheable results. The Session API's memory, versioned append, and turn-boundary compaction are all asserted, not described.

Two honest gaps. First, no real language model was called: the model is a stub, and the tool-calling behaviour of a real model — which tools it chooses, how it phrases the arguments — is the model's, not the framework's. Swap in an OpenAI or Ollama `ChatModel` bean and the wiring in this post does not change, but the *choices* are untested here. Second, the Java MCP client was verified over stdio, not over HTTP: the sandbox this ran in blocks outbound TCP from Java processes, so the HTTP wire was verified with `curl` reproducing exactly the requests the Java client sends — the same endpoints, the same headers, the same bodies — rather than by running the Java client itself. The server side binds and serves fine; it is the Java *client* socket that is blocked.

The migration notes, in one place: `ChatMemory` is deprecated, removal in 3.0.0 — move to the Session API and its `SessionMemoryAdvisor`; `defaultToolCallbacks(...)` is deprecated, use `defaultTools(...)`; the `SyncMcpToolCallbackProvider` constructors are deprecated, use `builder()`; `CallToolRequest` takes three arguments now; compile with `-parameters` or your tool schemas say `arg0`; and if your custom `ChatModel` returns plain `ChatOptions`, the tool loop will silently not run. That last one is the trap this post exists to save you from.

The weather server is a toy, but the shape is the real thing: any method you can annotate is a tool, any client that speaks MCP can call it, and the session remembers the conversation while the tools do the work. If the tool calls a downstream that can fail — and it will — the next thing to add is the resilience chain: a timeout, a backoff with jitter, and a circuit breaker, the same chain from [Retries Done Right](/posts/retries-backoff-jitter-spring-boot/). The model can ask for the weather; it is your job to make sure the answer survives a bad day.

## Verification note

Spring AI 2.1.0 GA confirmed against the Spring announcement (November 20, 2026); MCP 2026-07-28 behaviour confirmed against the MCP Java SDK 2.2.0 classes and the live wire transcripts above; the `STREAMABLE` default and the `ToolCallingAdvisor` skip rule confirmed against the framework source. The harness lives outside the blog repo: seven JUnit 6 tests (three session, three stdio-server, one full tool loop), a `verify_http.sh` curl script for the three HTTP protocols, and a curl-based Maven Central resolver — Maven itself cannot download through this machine's egress proxy, so dependencies were resolved with a small Python script against Central instead. No Docker was involved; nothing here was run against a live LLM or a live deployment.
