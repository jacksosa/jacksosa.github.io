---
title: Java, Spring Boot & AWS | Publisher-Subscriber Pattern Using SNS & SQS
tags: [ Java, AWS, Spring Boot, SNS, SQS ]
style: fill
color: info
description: As a Java developer, I’ve used AWS SNS and SQS to build scalable pub/sub systems. Here’s how to implement the publisher-subscriber pattern in Spring Boot for loosely coupled microservices.
---

# Implementing Pub/Sub with AWS SNS and SQS in Spring Boot: A Java Developer’s Guide

As a Java developer who’s built systems like Mosaic Smart Data’s real-time API pipeline, Co-op’s pricing analytics, ESG
Global’s smart metering orchestration, and Ribby Hall Village’s data warehouse, I’ve leaned on event-driven
architectures to create scalable, decoupled microservices. The publisher-subscriber (pub/sub) pattern, enabled by AWS
SNS and SQS, has been a go-to for asynchronous communication, ensuring systems evolve independently while handling
high-throughput data. In this guide, I’ll walk through implementing pub/sub in Spring Boot, using a user management
service as an example, drawing on my experience delivering robust backend solutions.

## Why SNS and SQS for Pub/Sub?

Unlike direct SQS or Kafka queues, which tie publishers to specific subscribers, SNS acts as a middleware, broadcasting
messages to multiple SQS queues without the publisher knowing who’s listening. This decoupling was critical in my Mosaic
Smart Data pipeline, where market data fed multiple analytics services. SNS simplifies scaling—add a new subscriber
queue, and the publisher remains unchanged. Below, I’ll show how to set up a publisher microservice sending user
creation events to an SNS topic and a subscriber consuming them from an SQS queue.

![AWS SNS SQS Spring Architecture Diagram]({{ site.baseurl }}/assets/images/posts/spring-cloud-sns-sqs-pubsub.png "AWS
SNS SQS Spring Architecture Diagram")

## Publisher Microservice Setup

The publisher simulates a user management service, exposing a REST API to create users and publishing events to an SNS
topic (`user-account-created`). I use Spring Cloud AWS to streamline AWS interactions, avoiding the complexity of raw
SDKs.

### Dependencies

Add Spring Cloud AWS SNS starter and BOM to your `pom.xml` for version compatibility:

```xml

<properties>
    <spring.cloud.version>3.1.1</spring.cloud.version>
</properties>
<dependencies>
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sns</artifactId>
</dependency>
</dependencies>
<dependencyManagement>
<dependencies>
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws</artifactId>
        <version>${spring.cloud.version}</version>
        <type>pom</type>
        <scope>import</scope>
    </dependency>
</dependencies>
</dependencyManagement>
```

### Configuration

Define AWS credentials and SNS region in `application.yaml`:

```yaml
spring:
  cloud:
    aws:
      credentials:
        access-key: ${AWS_ACCESS_KEY}
        secret-key: ${AWS_SECRET_KEY}
      sns:
        region: ${AWS_SNS_REGION}
co:
  uk:
    trinitylogic:
      aws:
        sns:
          topic-arn: ${AWS_SNS_TOPIC_ARN}
```

Map the SNS topic ARN to a POJO using `@ConfigurationProperties` for clean access:

```java
import lombok.Getter;
import lombok.Setter;

import javax.validation.constraints.NotBlank;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

@Getter
@Setter
@Validated
@ConfigurationProperties(prefix = "uk.co.trinitylogic.aws.sns")
public class AwsSnsTopicProperties {
    @NotBlank(message = "SNS topic ARN must be configured")
    private String topicArn;
}
```

### IAM Permissions

The publisher’s IAM user needs `sns:Publish` permission:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "sns-topic-arn"
    }
  ]
}
```

### Publishing Messages

The service publishes a trimmed user creation event to SNS:

```java
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import io.awspring.cloud.sns.core.SnsTemplate;

@Slf4j
@Service
@RequiredArgsConstructor
public class UserService {
    private final SnsTemplate snsTemplate;
    private final AwsSnsTopicProperties awsSnsTopicProperties;

    public void create(UserCreationRequestDto userCreationRequest) {
        // Save user to database
        var payload = removePassword(userCreationRequest);
        snsTemplate.convertAndSend(awsSnsTopicProperties.getTopicArn(), payload);
        log.info("Published message to topic ARN: {}", awsSnsTopicProperties.getTopicArn());
    }
}
```

Expose the API via a controller:

```java
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.Valid;

@RestController
@RequiredArgsConstructor
@RequestMapping("/api/v1/users")
public class UserController {
    private final UserService userService;

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<HttpStatus> createUser(@Valid @RequestBody UserCreationRequestDto userCreationRequest) {
        userService.create(userCreationRequest);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }
}
```

Test with a cURL request:

```bash
curl -X POST http://localhost:8080/api/v1/users \
-H "Content-Type: application/json" \
-d '{"name": "John Doe", "emailId": "john@trinitylogic.co.uk", "password": "secure123"}'
```

This logs a successful publish to the SNS topic, confirming the user creation event was sent.

````text
Successfully published message to topic ARN: <ARN-value-here>
````

## Subscriber Microservice Setup

The subscriber simulates a notification service, consuming user creation events from an SQS queue (
`dispatch-email-notification`) to log email dispatch actions. It uses Spring Cloud AWS to simplify SQS
interactions, similar to the publisher.

### Dependencies

Add the SQS starter to `pom.xml`:

```xml

<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sqs</artifactId>
</dependency>
```

### Configuration

Define SQS properties in `application.yaml`:

```yaml
spring:
  cloud:
    aws:
      credentials:
        access-key: ${AWS_ACCESS_KEY}
        secret-key: ${AWS_SECRET_KEY}
      sqs:
        region: ${AWS_SQS_REGION}

co:
  uk:
    trinitylogic:
      aws:
        sqs:
          queue-url: ${AWS_SQS_QUEUE_URL}
```

### Consuming Messages

Use `@SqsListener` to process messages:

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import io.awspring.cloud.sqs.annotation.SqsListener;

@Slf4j
@Component
public class EmailNotificationListener {
    @SqsListener("${co.uk.trinitylogic.aws.sqs.queue-url}")
    public void listen(UserCreatedEventDto userCreatedEvent) {
        log.info("Dispatching email to {} on {}", userCreatedEvent.getName(), userCreatedEvent.getEmailId());
        // Email dispatch logic
    }
}
```

To handle SNS metadata, enable raw message delivery or use `@SnsNotificationMessage`:

```java

@SqsListener("${co.uk.trinitylogic.aws.sqs.queue-url}")
public void listen(@SnsNotificationMessage UserCreatedEventDto userCreatedEvent) {
    log.info("Dispatching email to {} on {}", userCreatedEvent.getName(), userCreatedEvent.getEmailId());
}
```

### IAM Permissions

The subscriber’s IAM user needs `sqs:ReceiveMessage` and `sqs:DeleteMessage`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage"
      ],
      "Resource": "sqs-queue-arn"
    }
  ]
}
```

### SQS-SNS Subscription

Subscribe the SQS queue to the SNS topic via AWS Console or CLI. Add an SQS resource policy to allow SNS to send
messages:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "sqs-queue-arn",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "sns-topic-arn"
        }
      }
    }
  ]
}
```

## Encryption at Rest with AWS KMS

For sensitive data, like user details in my projects, encryption at rest is crucial for compliance (e.g., GDPR,
PCI-DSS). Create a KMS key and enable encryption on SNS and SQS. Update the KMS key policy:

```json
[
  {
    "Effect": "Allow",
    "Principal": {
      "Service": "sqs.amazonaws.com"
    },
    "Action": [
      "kms:GenerateDataKey",
      "kms:Decrypt"
    ],
    "Resource": "kms-key-arn",
    "Condition": {
      "ArnEquals": {
        "aws:SourceArn": "sqs-queue-arn"
      }
    }
  },
  {
    "Effect": "Allow",
    "Principal": {
      "Service": "sns.amazonaws.com"
    },
    "Action": [
      "kms:GenerateDataKey",
      "kms:Decrypt"
    ],
    "Resource": "kms-key-arn",
    "Condition": {
      "ArnEquals": {
        "aws:SourceArn": "sns-topic-arn"
      }
    }
  }
]
```

Add KMS permissions to the publisher’s IAM user:

```json
{
  "Effect": "Allow",
  "Action": [
    "kms:GenerateDataKey",
    "kms:Decrypt"
  ],
  "Resource": "kms-key-arn"
}
```

This ensures data security in transit and at rest, meeting compliance standards.

## Testing with LocalStack and Testcontainers

To validate the pub/sub flow, I use LocalStack and Testcontainers for integration testing. This setup mimics AWS
services locally, allowing for end-to-end testing without incurring costs or needing AWS credentials.

```xml

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>localstack</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.awaitility</groupId>
        <artifactId>awaitility</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

Create a `provision-resources.sh` script in `src/test/resources`:

```bash
#!/bin/bash
topic_name="user-account-created"
queue_name="dispatch-email-notification"
sns_arn_prefix="arn:aws:sns:us-east-1:000000000000"
sqs_arn_prefix="arn:aws:sqs:us-east-1:000000000000"

awslocal sns create-topic --name $topic_name
awslocal sqs create-queue --queue-name $queue_name
awslocal sns subscribe --topic-arn "$sns_arn_prefix:$topic_name" --protocol sqs --notification-endpoint "$sqs_arn_prefix:$queue_name"
echo "Successfully provisioned resources"
```

Set up the integration test:

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.testcontainers.containers.localstack.LocalStackContainer;
import org.testcontainers.utility.DockerImageName;
import org.testcontainers.utility.MountableFile;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.system.OutputCaptureExtension;
import org.springframework.boot.test.system.CapturedOutput;
import org.testcontainers.containers.wait.strategy.Wait;
import org.awaitility.Awaitility;

import java.util.concurrent.TimeUnit;

@SpringBootTest
class PubSubIT {
    @Autowired
    private MockMvc mockMvc;

    private static final LocalStackContainer localStackContainer;
    private static final String TOPIC_ARN = "arn:aws:sns:us-east-1:000000000000:user-account-created";
    private static final String QUEUE_URL = "http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/dispatch-email-notification";

    static {
        localStackContainer = new LocalStackContainer(DockerImageName.parse("localstack/localstack:3.3"))
                .withCopyFileToContainer(MountableFile.forClasspathResource("provision-resources.sh", 0744), "/etc/localstack/init/ready.d/provision-resources.sh")
                .withServices(LocalStackContainer.Service.SNS, LocalStackContainer.Service.SQS)
                .waitingFor(Wait.forLogMessage(".*Successfully provisioned resources.*", 1));
        localStackContainer.start();
    }

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.cloud.aws.credentials.access-key", localStackContainer::getAccessKey);
        registry.add("spring.cloud.aws.credentials.secret-key", localStackContainer::getSecretKey);
        registry.add("spring.cloud.aws.sns.region", localStackContainer::getRegion);
        registry.add("spring.cloud.aws.sns.endpoint", localStackContainer::getEndpoint);
        registry.add("co.uk.trinitylogic.aws.sns.topic-arn", () -> TOPIC_ARN);
        registry.add("spring.cloud.aws.sqs.region", localStackContainer::getRegion);
        registry.add("spring.cloud.aws.sqs.endpoint", localStackContainer::getEndpoint);
        registry.add("co.uk.trinitylogic.aws.sqs.queue-url", () -> QUEUE_URL);
    }

    @Test
    void test(CapturedOutput output) throws Exception {
        var name = "Jane Doe";
        var emailId = "jane@trinitylogic.co.uk";
        var userCreationRequestBody = String.format("""
                {
                    "name": "%s",
                    "emailId": "%s",
                    "password": "secure456"
                }
                """, name, emailId);

        mockMvc.perform(post("/api/v1/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(userCreationRequestBody))
                .andExpect(status().isCreated());

        var expectedPublisherLog = String.format("Published message to topic ARN: %s", TOPIC_ARN);
        Awaitility.await().atMost(1, TimeUnit.SECONDS).until(() -> output.getAll().contains(expectedPublisherLog));

        var expectedSubscriberLog = String.format("Dispatching email to %s on %s", name, emailId);
        Awaitility.await().atMost(1, TimeUnit.SECONDS).until(() -> output.getAll().contains(expectedSubscriberLog));
    }
}
```

This test, validates the entire pub/sub flow using MockMVC and Awaitility, ensuring messages flow from SNS to SQS.

## Conclusion

Implementing pub/sub with AWS SNS and SQS in Spring Boot creates a loosely coupled, scalable architecture, as I’ve done
for real-time systems like Mosaic Smart Data’s market data pipeline. Spring Cloud AWS simplifies integration, while KMS
encryption ensures security. LocalStack and Testcontainers make testing robust, mirroring my approach to reliable
deployments.

Try this setup in your next project and Share your wins or tips with me [here]({{ site.baseurl }}/contact/), I’d love to
hear your story!

<p class="text-center">
{% include elements/button.html link="https://docs.aws.amazon.com/sns/latest/dg/welcome.html" text="AWS SNS Docs" %}
{% include elements/button.html link="https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html" text="AWS SQS Docs" %}
</p>