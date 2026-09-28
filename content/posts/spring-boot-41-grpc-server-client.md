---
title: "Spring Boot 4.1's First-Class gRPC: Server + Client in One Project"
date: 2026-09-28T08:33:48-0400
draft: false
ShowToc: true
description: >-
  Spring Boot 4.1 moved gRPC auto-configuration into Boot itself: real
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

Adding gRPC to Spring Boot used to mean picking a third-party starter (LogNet, yidongnan), pinning `grpc-java` versions by hand, and writing your own server lifecycle glue. Spring gRPC 1.0 made it official but kept the auto-configuration outside Boot. **Spring Boot 4.1 finished the job**: the auto-configuration moved into Boot itself, the starters are on start.spring.io, and the BOM manages every `io.grpc` version. If you can build a REST endpoint in Spring, you already know how to build a gRPC service.

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

On start.spring.io they're just "gRPC Server" and "gRPC Client". The Boot BOM pins `spring-grpc-core:1.1.0` and `grpc-java:1.80.0` — you never declare versions yourself. Test variants exist too: `spring-boot-starter-grpc-server-test` and `spring-boot-starter-grpc-client-test` (more on those in Testing below).

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

message HelloRequest { string name = 1; }
message HelloReply { string message = 1; }
```

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

Any bean implementing `io.grpc.BindableService` is auto-exposed. In practice: extend the generated `ImplBase`, slap on `@GrpcService`, done. No server builder, no lifecycle code.

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
        // server-side streaming works exactly like the grpc-java API you already know
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

Boot starts a Netty gRPC server on 9090. Everything is configurable under `spring.grpc.server.*`: TLS via SSL bundles, keep-alive, message size limits, and graceful shutdown. Reflection and health services are registered automatically — point `grpcurl` at it and it just works:

```bash
grpcurl -plaintext localhost:9090 list
# com.example.demo.proto.Greeter
# grpc.health.v1.Health
# grpc.reflection.v1.ServerReflection
```

### Exception handling and interceptors

The programming model will feel familiar: `@GrpcAdvice` is the gRPC analog of `@RestControllerAdvice`, mapping exceptions to gRPC statuses, and `@GlobalServerInterceptor` beans apply to every service.

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

## The client: inject a stub like a RestClient

Declare the channel target in properties, import the clients, inject the stub. No `ManagedChannelBuilder` anywhere in your code.

```properties
spring.grpc.client.channel.greeter.address=localhost:9090
```

```java
@SpringBootApplication
@ImportGrpcClients("com.example.demo.proto")  // or target a specific stub class
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

The channel name (`greeter`) maps to `spring.grpc.client.channel.greeter.*` — address, TLS, keep-alive, retries all live there, configured like a data source.

## Testing: in-process, no ports

This is the nicest surprise. `@AutoConfigureTestGrpcTransport` swaps the Netty server for gRPC's in-process transport: your test still flows through interceptors, `@GrpcAdvice` handlers, and marshalling — just without TCP and without port conflicts. The annotation comes from the test starters — drop `spring-boot-starter-grpc-server-test` (or its client sibling) into your test scope and it's on the classpath.

```java
@SpringBootTest
@AutoConfigureTestGrpcTransport
class GreeterServiceTest {

    @Autowired
    GreeterGrpc.GreeterBlockingStub greeter;

    @Test
    void saysHello() {
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

Fast, deterministic, no `@DynamicPropertySource` port juggling. This alone is worth the upgrade.

## One port for both worlds

Running gRPC next to an existing REST API? Boot 4.1 supports a Servlet-embedded transport mode so gRPC and Spring MVC share one port over HTTP/2, instead of running the gRPC server on a separate Netty port. Useful for the migration period where half your clients still speak REST.

## When gRPC, when REST

Honest framing, since the hype oversells it:

| | gRPC | REST |
|---|---|---|
| Contract | Strong (proto, codegen) | Loose (OpenAPI if you're disciplined) |
| Payload | Protobuf binary — small, fast | JSON — human-readable, debuggable |
| Streaming | First-class (bidi, server, client) | Bolted on (SSE, websockets) |
| Browser clients | Needs grpc-web proxy | Native |
| Tooling | grpcurl, reflection | curl, every tool ever |

gRPC wins for service-to-service calls with a stable contract and for streaming. REST still wins for public APIs, browser clients, and anything where "curl it and read the response" matters. Boot 4.1 doesn't pick a side — it just makes the gRPC side as easy as the REST side always was.

## The interview one-liner

*"Spring Boot 4.1 moved gRPC auto-configuration into Boot itself: `@GrpcService` exposes any `BindableService`, `@ImportGrpcClients` injects type-safe stubs configured under `spring.grpc.client.channel.*`, and `@AutoConfigureTestGrpcTransport` gives you in-process tests. No third-party starter, no manual channel code."*
