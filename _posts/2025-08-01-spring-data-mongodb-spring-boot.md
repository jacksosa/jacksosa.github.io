---
title: "Spring Boot | Spring Data MongoDB — Modelling and Querying Documents"
date: 2026-08-01
tags: [Java, Spring Boot, MongoDB, Spring Data, NoSQL]
style: fill
color: success
description: >-
  Using Spring Data MongoDB in Spring Boot — @Document modelling, repositories, MongoTemplate for complex queries, aggregation pipelines, indexing, and avoiding the N+1 problem.
---

MongoDB is the right choice for certain problems: storing event streams, capturing variable-schema documents, handling high write throughput with flexible reads. At ESG Global we used it for smart meter reading storage — millions of time-series readings per day with varying metadata schemas depending on the meter type. At DWP Digital it was the event store backing our CQRS read models. Spring Data MongoDB makes the basic operations straightforward, but the query API and aggregation pipeline deserve more attention than most tutorials give them.

## Document Modelling

Spring Data MongoDB maps Java objects to BSON documents via `@Document`:

```java
@Document(collection = "benefit_claims")
public class ClaimDocument {

    @Id
    private String id;

    @Field("claim_ref")
    private String claimReference;

    private ClaimStatus status;
    private ClaimType type;

    @DBRef(lazy = true)
    private ClaimantDocument claimant;  // avoid — prefer embedding or ID references

    @Indexed
    private String claimantNiNumber;    // index for frequent query pattern

    private List<AssessmentDocument> assessments = new ArrayList<>();

    private Instant submittedAt;
    private Instant lastUpdatedAt;
}
```

A few design principles I follow:

**Prefer embedding over `@DBRef`.** `@DBRef` creates a reference to another document, requiring a separate query to load. Embedding includes related data in the same document — faster to read, atomic to write. Embed when the embedded data belongs exclusively to the parent and won't be queried independently.

**Model for your query patterns.** MongoDB has no joins. Design documents so that the queries you'll actually run against them work efficiently. If you often load a claim with its latest assessment, embed assessments in the claim document.

## Repositories — Simple Queries

For straightforward queries, `MongoRepository` generates implementations from method names:

```java
@Repository
public interface ClaimRepository extends MongoRepository<ClaimDocument, String> {

    List<ClaimDocument> findByStatus(ClaimStatus status);

    List<ClaimDocument> findByClaimantNiNumberAndStatus(
        String niNumber, ClaimStatus status);

    Optional<ClaimDocument> findByClaimReference(String ref);

    Page<ClaimDocument> findByStatusAndType(
        ClaimStatus status, ClaimType type, Pageable pageable);

    long countByStatus(ClaimStatus status);

    @Query("{ 'submittedAt': { $gte: ?0, $lt: ?1 } }")
    List<ClaimDocument> findSubmittedBetween(Instant from, Instant to);
}
```

The `@Query` annotation accepts native MongoDB query JSON when the derived method name becomes unwieldy.

## MongoTemplate — Complex Queries

`MongoTemplate` gives you full control over queries, updates, and document structure:

```java
@Service
public class ClaimQueryService {

    private final MongoTemplate mongoTemplate;

    public List<ClaimDocument> findStaleUnprocessedClaims(Duration staleDuration) {
        Instant cutoff = Instant.now().minus(staleDuration);

        Query query = new Query(
            Criteria.where("status").is(ClaimStatus.PENDING)
                .and("submittedAt").lt(cutoff)
                .and("assessments").size(0)
        );
        query.with(Sort.by(Sort.Order.asc("submittedAt")));
        query.limit(100);

        return mongoTemplate.find(query, ClaimDocument.class);
    }

    public UpdateResult updateStatusAndTimestamp(String claimId, ClaimStatus newStatus) {
        Query query = new Query(Criteria.where("_id").is(claimId));
        Update update = new Update()
            .set("status", newStatus)
            .set("lastUpdatedAt", Instant.now())
            .push("statusHistory", new StatusHistoryEntry(newStatus, Instant.now()));

        return mongoTemplate.updateFirst(query, update, ClaimDocument.class);
    }
}
```

Use `mongoTemplate.updateFirst()` and `mongoTemplate.updateMulti()` for targeted field updates — loading a document, modifying it in Java, and saving the whole document wastes network bandwidth and risks overwriting concurrent changes.

## Aggregation Pipelines

For analytics and complex transformations, MongoDB aggregation pipelines are the right tool:

```java
public List<ClaimCountByStatus> countClaimsByStatus(Instant since) {
    MatchOperation match = Aggregation.match(
        Criteria.where("submittedAt").gte(since)
    );

    GroupOperation group = Aggregation.group("status")
        .count().as("count")
        .first("status").as("status");

    SortOperation sort = Aggregation.sort(Sort.by(Sort.Order.desc("count")));

    Aggregation aggregation = Aggregation.newAggregation(match, group, sort);

    AggregationResults<ClaimCountByStatus> results = mongoTemplate.aggregate(
        aggregation, "benefit_claims", ClaimCountByStatus.class);

    return results.getMappedResults();
}

record ClaimCountByStatus(String status, long count) {}
```

Aggregation pipelines execute entirely in MongoDB — no data is shipped to the application until the final result. For large collections, this is orders of magnitude faster than loading documents and aggregating in Java.

## Indexing

Without indexes, queries on large collections perform full collection scans. Add indexes for every field that appears in frequent `find()` or aggregation `$match` stages:

```java
@Document(collection = "benefit_claims")
@CompoundIndex(name = "status_type_idx", def = "{'status': 1, 'type': 1}")
public class ClaimDocument {

    @Indexed
    private String claimantNiNumber;

    @Indexed(expireAfterSeconds = 7776000) // TTL index — auto-delete after 90 days
    private Instant archivedAt;
}
```

Or create indexes programmatically via `MongoTemplate`:

```java
@Component
public class MongoIndexInitialiser implements ApplicationRunner {

    private final MongoTemplate mongoTemplate;

    @Override
    public void run(ApplicationArguments args) {
        IndexOperations indexOps = mongoTemplate.indexOps("benefit_claims");
        indexOps.ensureIndex(new Index("claimantNiNumber", Sort.Direction.ASC));
        indexOps.ensureIndex(
            new CompoundIndexDefinition(
                new Document("status", 1).append("submittedAt", -1)
            ).named("status_submitted_idx")
        );
    }
}
```

Run `db.benefit_claims.explain("executionStats").find({...})` in the Mongo shell to verify queries are using indexes. A `COLLSCAN` in the winning plan means a missing index.

## ProTips

- **Avoid N+1 with `@DBRef`.** Loading a list of claims, then fetching each claimant via `@DBRef`, is exactly the N+1 problem. Embed the claimant data you need, or load claimants separately with an `$in` query.
- **Use projections.** If you only need `claimReference` and `status`, don't load the entire document. Use `Query.fields().include("claimReference").include("status")` with `MongoTemplate`.
- **Transactions are available but expensive.** MongoDB supports multi-document transactions in replica set mode, but they're slower than single-document operations. Design your schema so that most writes are atomic within a single document.
- **Monitor with `db.currentOp()`.** Slow queries in production are visible in `db.currentOp()` and the slow query log. Set `operationProfiling.slowOpThresholdMs: 100` in Mongo config to capture queries slower than 100ms.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
