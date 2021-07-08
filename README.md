# README

## Introduction

This demo is for implement grpc with reactor

Features :
1. Integration GRPC Interface with validator
2. Integration GRPC Server with unit test
3. Integration GRPC client with unit test (grpc mock) and live test

### Requirement
1. Spring boot 2.4.x
2. Grpc Dependency
3. Cucumber
4. Junit Jupiter

### Configuring GRPC Interface

```xml
<properties>
    <grpc-libs.version>1.0.0-2</grpc-libs.version>
</properties>


<dependencyManagement>
  <dependencies>
      <dependency>
          <groupId>simultan.team.grpc.libs</groupId>
          <artifactId>server-service-libs</artifactId>
          <version>${grpc-libs.version}</version>
      </dependency>
      <dependency>
          <groupId>simultan.team.grpc.libs</groupId>
          <artifactId>model-service-libs</artifactId>
          <version>${grpc-libs.version}</version>
      </dependency>
  </dependencies>
</dependencyManagement>    
```


Create proto file
```protobuf
syntax = "proto3";
option java_multiple_files = true;
package demo.grpc.reactive.server.proto;

option java_package = "demo.grpc.reactive.server.proto";
option java_outer_classname = "ExampleProto";

import "validation.proto";

message ExampleRequest {
  repeated string ids = 1[(validation.repeatMin) = 0];
}

message ExampleResponse {
  string id = 1;
  string value = 2;

}

service ExampleService {
  rpc handle(ExampleRequest) returns (stream ExampleResponse);
}
``` 

Execute maven command 

```
mvn clean install
```


### Configuring GRPC Server

Add this starter library to your project.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>simultan.team.grpc.libs</groupId>
      <artifactId>grpc-starter</artifactId>
      <version>1.0.0-2</version>
    </dependency>
  </dependencies>
</dependencyManagement>    
```

Put grpc service dependency

 ```xml
<dependencies>
  // must set interface dependency you create grpc interface  
  <dependency>
    <groupId>simultan.team.grpc</groupId>
    <artifactId>demo-grpc-interface</artifactId>
    <version>1.0.0-2</version>
    <scope>compile</scope>
  </dependency>

  // grpc dependency
  <dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-all</artifactId>
    <version>1.35.0</version>
  </dependency>
  <dependency>
    <groupId>simultan.team.grpc.libs</groupId>
    <artifactId>server-service</artifactId>
    <version>1.0.0-2</version>
  </dependency>
</dependencies>
```

Put annotation `@EnableGRPC` in main class application, for example

```java
@EnableGRPC
@SpringBootApplication
public class ServerServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(ServerServiceApplication.class, args);
	}
}

```

Implement grpc service 
```java
@Slf4j
@GRPCService
public class ExampleService extends ReactorExampleServiceGrpc.ExampleServiceImplBase {

  @Override
  public Flux<ExampleResponse> handle(Mono<ExampleRequest> requestMono) {
    return requestMono
        .doOnSuccess(request -> log.info("receive request {}", request.toString()))
        .flatMapMany(request -> Mono.just(ExampleResponse.newBuilder().build()));
  }
}
```

Configuring GRPC Service Implement

```java
@Configuration
@AutoConfigureBefore(GRPCConfiguration.class)
public class DemoReactiveServiceConfiguration {

  @Bean
  BindableService exampleServiceImplBase() {
    return new ExampleService();
  }
}
```

Setup Properties
```properties
simultan.team.libs.grpc.server.port=9099
```

#### Integration Test

Integration Test is for testing integration server grpc. The purpose of this test is for testing on the grpc server. the request will be processed with a predetermined response 

Create testing file 

```java
package demo.server.service;

import demo.grpc.reactive.server.proto.ExampleRequest;
import demo.grpc.reactive.server.proto.ReactorExampleServiceGrpc;
import demo.server.config.DemoReactiveServiceConfiguration;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.ImportAutoConfiguration;
import org.springframework.boot.test.context.SpringBootTest;
import reactor.test.StepVerifier;
import simultan.team.libs.grpc.server.config.GRPCConfiguration;

import java.time.Duration;
import java.util.Arrays;
import java.util.concurrent.TimeUnit;

@SpringBootTest(classes = TestingConfiguration.class, properties = {
    "simultan.team.libs.grpc.server.port=2000",
    "simultan.team.libs.grpc.server.shutdownWait=1000"
})
@ImportAutoConfiguration({GRPCConfiguration.class, DemoReactiveServiceConfiguration.class})
public class ExampleServiceIT {

  private ManagedChannel channel;

  @BeforeEach
  void setUp() {
    // Connect to the sever
    this.channel = ManagedChannelBuilder
        .forAddress("localhost", 2000).usePlaintext()
        .build();
  }

  @Test
  void whenRequestExample_thenShouldSuccess() {

    ReactorExampleServiceGrpc.ReactorExampleServiceStub stub =
            ReactorExampleServiceGrpc.newReactorStub(channel)
            .withDeadlineAfter(1000, TimeUnit.SECONDS);

    StepVerifier.create(stub.handle(ExampleRequest.newBuilder()
          .addAllIds(Arrays.asList("1", "2"))
          .build()).collectList())
        .expectSubscription().thenAwait()
        .expectNextMatches(exampleResponses -> {
          Assertions.assertEquals(2, exampleResponses.size());
          return true;
        })
        .expectComplete()
        .verify(Duration.ofSeconds(5));
  }
}

```

Run demo grpc server service 

### Configuring GRPC Client

Setup dependency

```xml
<dependency>
    <groupId>simultan.team.grpc.libs</groupId>
    <artifactId>model-service</artifactId>
    <version>1.0.0-2</version>
</dependency>
<dependency>
    <groupId>simultan.team.grpc</groupId>
    <artifactId>demo-grpc-interface</artifactId>
    <version>1.0.0-2</version>
</dependency>
```

Setup client service class implement

```java
@Slf4j
public class DefaultDemoClientService implements DemoClientService {

  private final ReactorExampleServiceStub stub;

  public DefaultDemoClientService(ReactorExampleServiceStub stub) {
    this.stub = stub;
  }

  @Override
  public Mono<List<String>> handle(List<String> ids) {

    Flux<ExampleResponse> flux = stub.handle(toMessage(ids));

    return flux.doOnNext(response -> log.info("receive response id {}", response.getId()))
        .map(ExampleResponse::getId)
        .collectList()
        .doOnError(throwable -> log.error("problem occurred {}", throwable.getMessage()));
  }

  private Mono<ExampleRequest> toMessage(List<String> ids) {
    return Mono.just(ExampleRequest.newBuilder()
        .addAllIds(ids)
        .build());
  }
}
```

Setup client configuration
```java
@Configuration
public class ClientConfiguration {

  @Bean
  DemoClientService demoClientService(ManagedChannel demoChannel) {
    ReactorExampleServiceGrpc.ReactorExampleServiceStub stub =
        ReactorExampleServiceGrpc.newReactorStub(demoChannel);
    return new DefaultDemoClientService(stub);
  }

  @Bean
  ManagedChannel demoChannel() {
    ManagedChannel channel = ManagedChannelBuilder
        .forAddress("localhost", 9099)
        .usePlaintext()
        .idleTimeout(3, TimeUnit.SECONDS)
        .keepAliveTimeout(5, TimeUnit.SECONDS)
        .build();
    return channel;
  }
}
```

### Testing

#### A. Unit Test
This unit test test aims to test components from the grpc client using junit5's grpcmock

Setup dependency
```xml
<grpcmock-junit5.version>0.5.4</grpcmock-junit5.version>

<dependency>
    <groupId>org.grpcmock</groupId>
    <artifactId>grpcmock-junit5</artifactId>
    <version>${grpcmock-junit5.version}</version>
</dependency>
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <version>3.3.17.RELEASE</version>
    <scope>test</scope>
</dependency>
```

Create file test

```java
package demo.grpc.reactive.service;

import demo.grpc.reactive.server.proto.ExampleRequest;
import demo.grpc.reactive.server.proto.ExampleResponse;
import demo.grpc.reactive.server.proto.ExampleServiceGrpc;
import demo.grpc.reactive.server.proto.ReactorExampleServiceGrpc;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.Status;
import io.grpc.StatusRuntimeException;
import org.grpcmock.GrpcMock;
import org.grpcmock.junit5.GrpcMockExtension;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import reactor.test.StepVerifier;

import java.util.Arrays;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

import static org.grpcmock.GrpcMock.*;
import static org.junit.jupiter.api.Assertions.assertEquals;


@ExtendWith(GrpcMockExtension.class)
public class DemoClientServiceTest {

    private ManagedChannel channel;

    private DefaultDemoClientService demoClientService;

    @BeforeEach
    void setup() {
        channel = ManagedChannelBuilder.forAddress("localhost", GrpcMock.getGlobalPort())
                .usePlaintext()
                .build();

        demoClientService = new DefaultDemoClientService(
                ReactorExampleServiceGrpc.newReactorStub(channel));
    }

    @AfterEach
    void cleanup() {
        Optional.ofNullable(channel).ifPresent(ManagedChannel::shutdownNow);
    }

    @Test
    void sendDemoClient_thenShouldSuccess() {
        List<String> ids = Arrays.asList("1", "2");
        ExampleRequest request = ExampleRequest.newBuilder()
                .addAllIds(ids)
                .build();

        List<ExampleResponse> responses =
                Arrays.asList(ExampleResponse.newBuilder()
                        .setId("1")
                        .setValue("Example 1")
                        .build(), ExampleResponse.newBuilder()
                        .setId("2")
                        .setValue("Example 2")
                        .build());

        stubFor(serverStreamingMethod(ExampleServiceGrpc.getHandleMethod())
                .willReturn(responses));

        StepVerifier.create(demoClientService.handle(ids))
            .expectSubscription()
            .thenAwait()
            .expectNextMatches(result -> {
                if(!result.equals(responses.stream()
                        .map(ExampleResponse::getId)
                        .collect(Collectors.toList()))) {
                    throw new IllegalStateException("result not match");
                }

                return true;
            })
            .verifyComplete();

        verifyThat(calledMethod(ExampleServiceGrpc.getHandleMethod())
                .withRequest(request), times(1));
    }

    @Test
    void sendDemoClient_thenShouldAborted() {
        List<String> ids = Arrays.asList("1", "2");
        ExampleRequest request = ExampleRequest.newBuilder()
                .addAllIds(ids)
                .build();

        stubFor(serverStreamingMethod(ExampleServiceGrpc.getHandleMethod())
                .willReturn(Status.ABORTED));

        StepVerifier.create(demoClientService.handle(ids))
                .expectSubscription()
                .thenAwait()
                .expectErrorMatches(throwable -> {
                    StatusRuntimeException exception = (StatusRuntimeException) throwable;
                    assertEquals(exception.getStatus(), Status.ABORTED);
                    return true;
                });

        verifyThat(calledMethod(ExampleServiceGrpc.getHandleMethod())
                .withRequest(request), times(0));
    }
}
```

#### B. Live Test with cucumber
This live test aims to test components starting from the grpc client to being accepted by the grpc server and receiving a response

Testing Step
1. Run GRPC Server
2. Setup Dependency and file test
3. Run file testing

Go to Implementation
1. Run GRPC Server 
   <br />In Live test, you must run GRPC server before
2. Setup Dependency and file test

```xml
<cucumber.version>6.8.0</cucumber.version>

<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-core</artifactId>
    <version>${cucumber.version}</version>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-java</artifactId>
    <version>${cucumber.version}</version>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit</artifactId>
    <version>${cucumber.version}</version>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-spring</artifactId>
    <version>${cucumber.version}</version>
</dependency>
```

Create file scenario demo.feature

```text
Feature: try to test demo grpc client
  Scenario: call test demo grpc service integration
    When the service calls to test demo grpc
```

Create Root Testing file
```java
@RunWith(Cucumber.class)
@CucumberOptions(features = "src/test/resources/integration/demo")
@CucumberContextConfiguration
@SpringBootTest(classes = {ClientServiceApplication.class, TestingConfiguration.class},
    webEnvironment = WebEnvironment.DEFINED_PORT)
@TestPropertySource(locations = {"classpath:integration/demo/application.yml"})
public class DemoServiceTestingLive {

}
```

Create Step Testing file 
```java
package demo.grpc.reactive.service.integration;

import demo.grpc.reactive.service.DemoClientService;
import io.cucumber.java.en.When;
import org.springframework.beans.factory.annotation.Autowired;

import java.util.Arrays;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertNotNull;

public class StepDemoIntegration {

    @Autowired
    private DemoClientService service;

    @When("^the service calls to test demo grpc")
    public void the_service_calls_to_test_demo_grpc() throws Throwable {
        List<String> send = service.handle(Arrays.asList("1", "2")).block();
        assertNotNull(send);
    }
}
```

Run Root Testing file DemoServiceTestingLive.java

ENJOY!!!