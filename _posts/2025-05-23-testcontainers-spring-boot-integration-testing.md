---
title: "Testing | Testcontainers for Spring Boot — Real Databases, Real Confidence"
date: 2026-05-23
tags: [Java, Spring Boot, Testcontainers, Testing, PostgreSQL, MongoDB]
style: fill
color: secondary
description: >-
  Why Testcontainers beats mocked repositories for integration testing, and how to use it in Spring Boot with PostgreSQL, MongoDB, and Kafka — with real container setup and test speed tips.
---

I got burned by mocked repository tests at DWP Digital. A service had 95% test coverage, every test green, and a production deployment that failed because the Mongo query our team had written didn't behave the way the mock said it would. The mock had been configured to return data in a specific order; the real MongoDB returned it in a different order under load. An entire sprint of bug investigation traced back to a mocked dependency that lied.

Since then, I use Testcontainers for anything that touches a database, message broker, or external system. The tests are slower, but they tell the truth.

## The Core Idea

Testcontainers starts real Docker containers as part of your test lifecycle. Your tests connect to a genuine PostgreSQL instance, a genuine MongoDB replica set, a genuine Kafka broker — not a fake. When the tests finish, the containers are torn down. The feedback is accurate because the environment is accurate.

The dependency is straightforward:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <scope>test</scope>
</dependency>
```

## Setting Up PostgreSQL Integration Tests

With Spring Boot 3.1+, you can declare container beans directly in your test configuration:

```java
@SpringBootTest
@Testcontainers
class ClaimRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("claims_test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private ClaimRepository claimRepository;

    @Test
    void shouldFindClaimsByStatus() {
        claimRepository.save(new Claim("CLM-001", ClaimStatus.PENDING));
        claimRepository.save(new Claim("CLM-002", ClaimStatus.APPROVED));

        List<Claim> pending = claimRepository.findByStatus(ClaimStatus.PENDING);

        assertThat(pending).hasSize(1)
            .extracting(Claim::getReference)
            .containsExactly("CLM-001");
    }
}
```

`@DynamicPropertySource` wires the container's actual JDBC URL into the Spring context before beans are initialised. Spring then creates the `DataSource` pointing at the real container.

## Reusing Containers Across Tests

Starting a container per test class adds meaningful time. Use Testcontainers' static container pattern with Spring Boot's service connection to reuse containers across the test suite:

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestContainersConfig {

    @Bean
    @ServiceConnection // Spring Boot 3.1+ auto-wires datasource properties
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }

    @Bean
    @ServiceConnection
    MongoDBContainer mongoContainer() {
        return new MongoDBContainer("mongo:7");
    }

    @Bean
    @ServiceConnection
    KafkaContainer kafkaContainer() {
        return new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));
    }
}
```

With `@ServiceConnection`, Spring Boot automatically configures the datasource, MongoDB client, and Kafka properties from the running containers. No `@DynamicPropertySource` needed. Import this configuration into your test classes:

```java
@SpringBootTest
@Import(TestContainersConfig.class)
class ClaimServiceIntegrationTest {
    // Spring context wired to real containers
}
```

Declare the containers as static beans at the `@TestConfiguration` level and they're shared across the entire test run rather than restarted per class.

## Testing Kafka Consumers

Testing a Kafka consumer integration requires producing a message and asserting the downstream effect:

```java
@SpringBootTest
@Import(TestContainersConfig.class)
class BenefitClaimConsumerIntegrationTest {

    @Autowired
    private KafkaTemplate<String, BenefitClaimEvent> kafkaTemplate;

    @Autowired
    private ClaimRepository claimRepository;

    @Test
    void shouldProcessClaimEventAndPersist() throws Exception {
        BenefitClaimEvent event = new BenefitClaimEvent("CLM-TEST-001", ClaimType.HOUSING);

        kafkaTemplate.send("benefit.claims", event.getClaimId(), event).get();

        // Poll with timeout — consumer is async
        await().atMost(10, SECONDS).untilAsserted(() -> {
            Optional<Claim> claim = claimRepository.findByReference("CLM-TEST-001");
            assertThat(claim).isPresent()
                .get()
                .extracting(Claim::getStatus)
                .isEqualTo(ClaimStatus.PENDING);
        });
    }
}
```

The `await()` from Awaitility handles the asynchronous nature of Kafka consumption without introducing arbitrary `Thread.sleep()` calls.

## Container Startup Speed

Testcontainers tests are slower than unit tests — accept that and manage it. A few techniques keep the time reasonable:

**Ryuk and container reuse:** Testcontainers includes a Ryuk container for cleanup. In CI, set `TESTCONTAINERS_RYUK_DISABLED=true` if containers are cleaned up by your CI system anyway.

**Image caching:** Pre-pull images in your CI pipeline before tests run. Docker image pull time is often the biggest overhead.

**Test slicing:** Not every test needs the full Spring context and all containers. A `@DataJpaTest` slice loads only the JPA layer. A `@WebMvcTest` slice loads only the web layer with a mock service layer. Reserve full `@SpringBootTest` + Testcontainers for genuine integration scenarios.

## ProTips

- **`@Testcontainers` + `@Container` on a static field** means the container starts once per class. If you need a fresh database state per test method, use `@Transactional` on test methods to roll back after each test — far cheaper than restarting the container.
- **Test MongoDB aggregation pipelines with real data.** Mongo aggregation behaviour varies by version and index state. This is exactly the class of bug that mocks hide.
- **Use `@DataJpaTest` with Testcontainers for repository-only tests.** It loads only the JPA and Liquibase/Flyway layers, making the context fast while still connecting to a real database.
- **Add Testcontainers to your CI matrix.** Most CI platforms support Docker. The few extra minutes per build are worth the bug class they eliminate.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
