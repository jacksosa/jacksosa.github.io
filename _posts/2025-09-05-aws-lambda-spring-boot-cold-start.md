---
title: "AWS | AWS Lambda with Spring Boot — Cold Start Optimisation"
tags: [Java, AWS, Lambda, Spring Boot, Serverless]
style: fill
color: danger
description: Deploying Spring Boot to AWS Lambda and tackling cold start latency — GraalVM native image, SnapStart, tiered compilation, and knowing when Lambda is the wrong choice for Java workloads.
---

AWS Lambda is a natural fit for event-driven architectures — and it's where I've deployed several components of financial data ingestion pipelines and AWS-hosted services. The friction comes from Java's cold start time. A standard Spring Boot application with its full context can take 8–15 seconds to initialise on a fresh Lambda container. For APIs where users experience that latency, it's unacceptable. The good news is there are several techniques that bring cold starts to under 1 second, and in many cases the choice of approach is straightforward.

## Why Java Cold Starts Are Slow

Lambda cold start time breaks down into:
1. Container provisioning (AWS overhead, ~200ms — not your problem)
2. JVM startup and class loading
3. Spring ApplicationContext initialisation — component scanning, auto-configuration, bean creation
4. Your application's startup logic

Step 3 is the main culprit. Spring Boot on the JVM needs to load and initialise hundreds of beans, run auto-configuration, and connect to external dependencies. On a warmed Lambda with an already-running JVM, this cost is zero. On a cold start, you pay it in full.

## Baseline Measurement

Before optimising, measure. Add the AWS Lambda Powertools Java dependency for structured metrics:

```java
public class ClaimHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    private final ClaimService claimService;

    public ClaimHandler() {
        // This constructor runs at cold start
        long start = System.currentTimeMillis();
        this.claimService = SpringLambda.getBean(ClaimService.class);
        long initTime = System.currentTimeMillis() - start;
        log.info("Cold start init time: {}ms", initTime);
    }

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent event, Context context) {
        // Warm invocations arrive here directly
        return claimService.handle(event);
    }
}
```

Log and track `initTime`. Track in CloudWatch and alert when p99 cold start time exceeds your threshold.

## SnapStart — Fastest Path to Improvement

AWS Lambda SnapStart (available for Java 11+) snapshots the Lambda execution environment after initialisation and restores from the snapshot on subsequent cold starts. For Spring Boot applications, this means you pay the initialisation cost once — future cold starts restore from the snapshot in ~200ms.

Enable it in your `template.yml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  ClaimFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: java21
      SnapStart:
        ApplyOn: PublishedVersions
      Handler: com.example.ClaimHandler
      CodeUri: target/claim-service.jar
      MemorySize: 512
      Timeout: 30
```

SnapStart requires that your initialisation code is idempotent — the snapshot is taken once and restored many times. Watch for:
- Random values or UUIDs generated at init time (cached incorrectly across restorations)
- External connections opened at init time (restored connections may be stale)

Implement `CRaC` hooks to handle pre-checkpoint and post-restore lifecycle:

```java
import org.crac.Context;
import org.crac.Core;
import org.crac.Resource;

@Component
public class SnapStartLifecycle implements Resource {

    private final DataSource dataSource;

    @PostConstruct
    public void register() {
        Core.getGlobalContext().register(this);
    }

    @Override
    public void beforeCheckpoint(Context<? extends Resource> ctx) throws Exception {
        // Close connections before snapshot is taken
        dataSource.getConnection().close();
    }

    @Override
    public void afterRestore(Context<? extends Resource> ctx) throws Exception {
        // Re-establish connections after restore
        dataSource.getConnection(); // warm up connection pool
    }
}
```

## GraalVM Native Image

GraalVM compiles your Spring Boot application to a native executable — no JVM startup, AOT-compiled code. Cold starts of 50–200ms are achievable:

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <configuration>
        <imageName>claim-function</imageName>
        <buildArgs>
            <buildArg>--no-fallback</buildArg>
            <buildArg>--enable-url-protocols=https</buildArg>
        </buildArgs>
    </configuration>
</plugin>
```

Spring Boot 3.x has first-class GraalVM support via `spring-boot-starter` + AOT processing. Most Spring features work — but dynamic features (reflection, proxies, runtime classpath scanning) require explicit hint configuration:

```java
@RegisterReflectionForBinding({ClaimEvent.class, ClaimResponse.class})
@Configuration
public class LambdaConfig {
    // beans for Lambda handler
}
```

Native image compilation is slow (5–10 minutes on a typical build machine) and imposes constraints on dynamic features. For Lambda functions with stable, well-defined inputs and outputs, it's the best option for cold start performance.

## Tiered Compilation Flags

Without native image, JVM flags reduce cold start time:

```bash
# In Lambda function environment variables
JAVA_TOOL_OPTIONS="-XX:+TieredCompilation -XX:TieredStopAtLevel=1"
```

`TieredStopAtLevel=1` disables C2 (optimising) compilation and uses only the fast interpreter + C1 (baseline) compiler. Startup is faster; peak throughput is lower. For Lambda where invocations are short-lived, this is generally the right trade-off.

## Memory Configuration

Lambda CPU allocation scales with memory. A 512MB Lambda gets half a vCPU; 1024MB gets a full vCPU; 1769MB gets two vCPUs. For a Spring Boot application:

- Cold start time at 256MB: ~6s
- Cold start time at 512MB: ~3s  
- Cold start time at 1024MB: ~1.5s
- Cold start time at 2048MB: ~800ms

Extra memory reduces cold start time faster than the marginal cost increase. For a Lambda behind an API Gateway, 1024–2048MB is often optimal.

## When Lambda Is the Wrong Choice for Java

Lambda is not appropriate for:
- **Sustained high-throughput workloads.** Lambda's per-invocation billing and cold start overhead make it expensive and latency-variable compared to a sized ECS Fargate service.
- **Long-running Kafka consumers.** Lambda has a maximum execution time of 15 minutes. A continuous consumer belongs on ECS or EKS.
- **Applications with tight p99 latency requirements.** Cold starts on Lambda are non-deterministic. If your SLA requires consistent sub-100ms p99, use a persistently-running service.

Lambda is ideal for: event-driven processing triggered by S3, SQS, SNS, EventBridge; infrequent API endpoints; scheduled batch jobs; and data transformation functions.

## ProTips

- **Start with SnapStart.** It's the lowest-effort, highest-impact improvement for most Spring Boot Lambdas. Enable it, measure cold starts, and decide if further optimisation is needed.
- **Use Spring's functional bean definition style.** `BeanDefinitionCustomizerParameterizedCondition` and `@Bean`-based configuration start faster than classpath component scanning.
- **Measure warm invocations too.** A Lambda with a 2-second cold start and a 100ms warm invocation has a very different cost/performance profile than one with a 200ms cold start and a 2-second warm invocation.
- **Set reserved concurrency on critical Lambdas.** Reserved concurrency keeps containers warm and bounds cold start exposure for latency-sensitive paths.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
