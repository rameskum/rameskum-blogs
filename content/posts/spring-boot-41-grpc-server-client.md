---
title: "Spring Boot 4.1's First-Class gRPC: Server + Client in One Project"
date: 2026-09-28T08:33:48-0400
draft: false
ShowToc: true
description: >-
  Spring Boot 4.1 brings first-class gRPC support to Boot itself: official
  starters on start.spring.io, @GrpcService servers, injected type-safe
  clients, and in-process test transport. From .proto to a working
  server-plus-client project, with the test and exception-handling pieces.
tags:
  - spring-boot
  - java
  - grpc
  - backend
  - microservices
  - interview
categories: article
keywords:
  - spring boot grpc
  - spring boot 4.1 grpc starter
  - grpc server client spring
---

Adding gRPC to Spring Boot traditionally meant choosing a third-party starter such as LogNet or yidongnan, managing gRPC versions, and dealing with server configuration yourself. Spring gRPC 1.0 made it official but kept the auto-configuration outside Boot. **Spring Boot 4.1 finished the job**: the auto-configuration moved into Boot itself, the starters are on start.spring.io, and the BOM manages the versions of the gRPC dependencies it provides. If you're comfortable building REST endpoints with Spring, Spring Boot 4.1 makes the gRPC programming model feel surprisingly familiar.

## The starters

Two dependencies, no third-party anything, no extra BOM:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-grpc-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-grpc-client</artifactId>
</dependency>
```

On start.spring.io they're just "gRPC Server" and "gRPC Client". The Spring Boot BOM manages the compatible Spring gRPC and gRPC Java versions, so you don't need to declare them yourself.

## Contract-first: the .proto

Proto files go in `src/main/proto`. Put your `.proto` files there, then use the protobuf Maven plugin to generate the Java classes:

```proto
// src/main/proto/greeter.proto
syntax = "proto3";

option java_package = "com.example.demo.proto";
option java_multiple_files = true;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
  rpc StreamHellos (HelloRequest) returns (stream HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

The Spring Boot gRPC starters provide the runtime pieces. The protobuf Maven plugin remains responsible for turning the `.proto` contract into generated Java classes during the build:

```xml
<!-- the standard protobuf-maven-plugin setup -->
<build>
  <extensions>
    <extension>
      <groupId>kr.motd.maven</groupId>
      <artifactId>os-maven-plugin</artifactId>
      <version>1.7.1</version>
    </extension>
  </extensions>
  <plugins>
    <plugin>
      <groupId>org.xolstice.maven.plugins</groupId>
      <artifactId>protobuf-maven-plugin</artifactId>
      <version>0.6.1</version>
      <configuration>
        <protocArtifact>com.google.protobuf:protoc:${os.detected.classifier}:exe:${protobuf.version}</protocArtifact>
        <pluginId>grpc-java</pluginId>
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.80.0:exe:${os.detected.classifier}</pluginArtifact>
      </configuration>
      <executions>
        <execution><goals><goal>compile</goal><goal>compile-custom</goal></goals></execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

## The server: one annotation

Any Spring bean implementing `io.grpc.BindableService` can be exposed as a gRPC service. In practice, extend the generated `ImplBase` and annotate it with `@GrpcService`. No server builder, no lifecycle code.

```java
@GrpcService
public class GreeterService extends GreeterGrpc.GreeterImplBase {

    @Override
    public void sayHello(HelloRequest request, StreamObserver<HelloReply> responseObserver) {
        HelloReply reply = HelloReply.newBuilder()
                .setMessage("Hello, " + request.getName())
                .build();
        responseObserver.onNext(reply);
        responseObserver.onCompleted();
    }

    @Override
    public void streamHellos(HelloRequest request, StreamObserver<HelloReply> responseObserver) {
        // standard gRPC Java server-streaming: send multiple responses through the StreamObserver
        for (int i = 1; i <= 3; i++) {
            responseObserver.onNext(HelloReply.newBuilder()
                    .setMessage("Hello #%d, %s".formatted(i, request.getName()))
                    .build());
        }
        responseObserver.onCompleted();
    }
}
```

```properties
# application.properties
spring.grpc.server.port=9090
```

Boot starts a Netty gRPC server on 9090. Everything is configurable under `spring.grpc.server.*`: TLS via SSL bundles, keep-alive, message size limits, and graceful shutdown. When `grpc-services` is on the classpath, Spring Boot can automatically configure gRPC reflection and health support — so `grpcurl` can discover the service without additional server configuration:

```bash
grpcurl -plaintext localhost:9090 list
# com.example.demo.proto.Greeter
# grpc.health.v1.Health
# grpc.reflection.v1.ServerReflection
```

`-plaintext` is right for this sample because the server doesn't use TLS — don't copy that flag into production. Each of these lines is a service you can then call with `grpcurl`: `grpc.health.v1.Health` for health checks, `grpc.reflection.v1.ServerReflection` for discovery.

### Exception handling and interceptors

The programming model will feel familiar: `@GrpcAdvice` is the gRPC equivalent of the familiar `@RestControllerAdvice` pattern, mapping exceptions to gRPC statuses, and `@GlobalServerInterceptor` beans apply to every service.

```java
@GrpcAdvice
public class GreeterExceptionHandler {

    @GrpcExceptionHandler(IllegalArgumentException.class)
    public StatusRuntimeException handleIllegalArgument(IllegalArgumentException ex) {
        return Status.INVALID_ARGUMENT.withDescription(ex.getMessage()).asRuntimeException();
    }
}

@Component
@GlobalServerInterceptor
public class LoggingInterceptor implements ServerInterceptor {
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call, Metadata headers, ServerCallHandler<ReqT, RespT> next) {
        log.info("gRPC call: {}", call.getMethodDescriptor().getFullMethodName());
        return next.startCall(call, headers);
    }
}
```

Multiple global interceptors can be ordered with Spring's usual `@Order` mechanism.

## The client: inject a stub like a RestClient

Declare the channel target in properties, import the clients, inject the stub. No `ManagedChannelBuilder` anywhere in your code.

```properties
spring.grpc.client.channel.greeter.target=localhost:9090
```

```java
@SpringBootApplication
@ImportGrpcClients(target = "greeter", types = GreeterGrpc.GreeterBlockingStub.class)
public class DemoApplication { ... }

@Service
public class GreetingClient {
    private final GreeterGrpc.GreeterBlockingStub greeter;

    public GreetingClient(GreeterGrpc.GreeterBlockingStub greeter) {
        this.greeter = greeter;
    }

    public String greet(String name) {
        HelloReply reply = greeter.sayHello(HelloRequest.newBuilder().setName(name).build());
        return reply.getMessage();
    }
}
```

The channel mapping is explicit now: `target = "greeter"` names the channel, and the channel's settings live under that name:

```
@ImportGrpcClients(target = "greeter")
                    ↓
spring.grpc.client.channel.greeter.target
                    ↓
localhost:9090
```

TLS, keep-alive, message-size limits, and other channel-specific settings can be configured under `spring.grpc.client.channel.greeter.*` too — configured like a data source. Retry-related channel configuration can also be supplied there where supported.

## Testing: in-process, no ports

This is the nicest surprise. `@AutoConfigureTestGrpcTransport` configures an in-process gRPC test transport, so the test doesn't need to open a TCP port. The test still exercises the Spring gRPC application layer, including service invocation and exception handling, without requiring a network listener. It doesn't replace tests that need to exercise the real network transport, TLS, HTTP/2 behavior, or deployment configuration. The annotation ships in the gRPC test starters — add `spring-boot-starter-grpc-server-test` (or its client sibling) to your test scope and it's on the classpath.

```java
@SpringBootTest
@AutoConfigureTestGrpcTransport
class GreeterServiceTest {

    @Autowired
    GreeterGrpc.GreeterBlockingStub greeter;

    @Test
    void returnsGreeting() {
        HelloReply reply = greeter.sayHello(HelloRequest.newBuilder().setName("Ramesh").build());
        assertThat(reply.getMessage()).isEqualTo("Hello, Ramesh");
    }

    @Test
    void mapsBadInputToInvalidArgument() {
        // exception-handler behavior, verified without a socket
        assertThatThrownBy(() -> greeter.sayHello(HelloRequest.newBuilder().setName("").build()))
                .isInstanceOf(StatusRuntimeException.class)
                .satisfies(e -> assertThat(((StatusRuntimeException) e).getStatus().getCode())
                        .isEqualTo(Status.Code.INVALID_ARGUMENT));
    }
}
```

Fast, deterministic, and no `@DynamicPropertySource` port juggling. For me, this is one of the nicest improvements in the new gRPC support.

## One port for both worlds

Running gRPC next to an existing REST API? Boot 4.1 supports a Servlet-embedded transport mode so gRPC and Spring MVC share one port over HTTP/2, instead of running the gRPC server on a separate Netty port. Useful for the migration period where half your clients still speak REST. The default standalone setup still uses a native gRPC server such as Netty — the Servlet transport is the opt-in for sharing the application's HTTP/2 servlet server.

## When gRPC, when REST

Honest framing, since the hype oversells it:

| | gRPC | REST |
|---|---|---|
| Contract | Explicit schema (.proto, codegen) | HTTP resources + optional OpenAPI contract |
| Payload | Binary Protobuf — typically compact and efficient | Text-based JSON — easy to inspect manually |
| Streaming | Built into the RPC model | Usually handled with SSE, WebSockets, or other HTTP mechanisms |
| Browser clients | Usually requires gRPC-Web or a gateway | Native HTTP/JSON APIs |
| Tooling | grpcurl, reflection | curl, every tool ever |

gRPC is often a strong fit for service-to-service communication, strongly typed contracts, and streaming RPCs. REST remains a natural fit for public HTTP APIs, browser-facing applications, and APIs where human-readable requests and responses are important. Boot 4.1 doesn't force either model — it just makes the gRPC side as easy as the REST side always was.

## The interview one-liner

*"Spring Boot 4.1 moved gRPC auto-configuration into Boot itself: `@GrpcService` exposes any `BindableService`, `@ImportGrpcClients` registers generated type-safe stubs as Spring beans configured under `spring.grpc.client.channel.*`, and `@AutoConfigureTestGrpcTransport` gives you in-process tests. No third-party gRPC starter, no manual channel code."*
