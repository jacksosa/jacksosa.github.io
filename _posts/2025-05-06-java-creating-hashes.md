---
title: Java | Hashing for Secure and Efficient Code
tags: [ Java, Hashing, Security, Clean Code, Spring Boot ]
style: fill
color: secondary
description: As a Java developer, I’ve used hashing to secure data in projects like ESG Global’s BOL Engine. Here’s my guide to creating hashes in Java, with real-world tips.
---

As a Java developer who’s built systems like Mosaic Smart Data’s real-time API pipeline, Co-op’s competitor pricing
reports, ESG Global’s BOL Engine, and Ribby Hall Village’s data warehouse, I’ve learned that hashing is a Swiss Army
knife for security and performance. Early on, I fumbled with checksums in Co-op’s pricing system, generating
inconsistent hashes that broke data lookups. Once I got the hang of Java’s hashing tools, I used them to secure user
data in ESG’s engine and speed up trade ID lookups in Mosaic’s pipeline. Hashing is critical for everything from
passwords to data integrity, and Java makes it straightforward. Here’s my guide to creating hashes in Java, packed with
examples from my projects and lessons I’ve learned the hard way.

## What Is a Hash Function?

A hash function takes any input—text, numbers, even files—and spits out a fixed-size string called a hash or digest.
It’s like a fingerprint: unique (mostly), deterministic (same input, same output), and fast. In ESG’s BOL Engine, I
hashed user IDs to securely store credentials. In Mosaic’s pipeline, I used hashes to verify trade event integrity. Hash
functions are one-way (you can’t reverse them) and aim for minimal collisions (different inputs producing the same
hash).

**ProTip**: Always choose a hash function that matches your use case, speed for checksums, security for passwords.

## Use Cases for Hash Functions

Hashing pops up everywhere in my projects:

- **Data Integrity**: In Mosaic’s pipeline, I hashed trade events to ensure data wasn’t tampered with during Kafka
  streaming.
- **Password Storage**: In ESG’s system, I hashed user passwords to keep them secure.
- **Data Lookup**: In Co-op’s pricing reports, I used hashes as keys in maps for fast price lookups.
- **Digital Signatures**: In Ribby Hall’s sync, I hashed config data to verify authenticity.

## Hashing Types

We will look at the following types of hash in this post:

1. **MD5** Message Digest: Fast but insecure (collisions found). I used it for quick checksums in Ribby Hall’s
   non-sensitive sync tasks.
2. **SHA** Secure Hash Algorithm: Secure and widely used. I relied on it for trade event integrity in Mosaic’s pipeline.
3. **PBKDF2** Password-Based Key Derivative Function with Hmac-SHA1 (PBKDF2WithHmacSHA1): Slow by design, perfect for
   passwords. I used it in ESG’s user authentication.

## Hashing with Java’s MessageDigest

Java’s `java.security.MessageDigest` is the go-to for MD5 and SHA:

````java
import java.security.MessageDigest;
import java.math.BigInteger;

public String hashTradeId(String tradeId) throws NoSuchAlgorithmException {
    MessageDigest md = MessageDigest.getInstance("SHA-256");
    byte[] digest = md.digest(tradeId.getBytes());
    return String.format("%064x", new BigInteger(1, digest));
}
````

Used like:

````java
String tradeId = "TRX12345";
String hash = hashTradeId(tradeId);
System.out.println(hash); // e.g., 2c26b46b68ffc68ff99b453c1d30413413422d706483bfa0f98a5e886266e7ae
````

**ProTip**: Always handle `NoSuchAlgorithmException` to avoid crashes if the algorithm isn’t supported.

## Hashing with Apache Commons Codec

Apache Commons Codec simplifies hashing with readable output formats:

````java
import org.apache.commons.codec.digest.DigestUtils;

public String hashPrice(String priceData) {
    return DigestUtils.sha256Hex(priceData);
}
````

Used like:

````java
String priceData = "19.99:GBP";
String hash = hashPrice(priceData);
System.out.println(hash); // e.g., 8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4
````

Add the dependency:

````xml
<dependency>
    <groupId>commons-codec</groupId>
    <artifactId>commons-codec</artifactId>
    <version>1.17.1</version>
</dependency>
````

## Hashing with Guava

Google’s Guava library offers robust hashing utilities:

````java
import com.google.common.hash.Hashing;

import java.nio.charset.StandardCharsets;

public String hashConfig(String config) {
    return Hashing.sha256().hashString(config, StandardCharsets.UTF_8).toString();
}
````

Used like:

````java
String config = "data.db:1000";
String hash = hashConfig(config);
System.out.println(hash); // e.g., 5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8
````

Add the dependency:

````xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>33.3.1-jre</version>
</dependency>
````

## Password Hashing with bcrypt

For passwords, bcrypt is my pick due to its adaptive design:

````java
import org.mindrot.jbcrypt.BCrypt;

public String hashPassword(String password) {
    return BCrypt.hashpw(password, BCrypt.gensalt(12));
}

public boolean checkPassword(String password, String hashed) {
    return BCrypt.checkpw(password, hashed);
}
````

Used like:

````java
String password = "user123";
String hashed = hashPassword(password);
System.out.println(hashed); // e.g., $2a$12$WqX8z7kZ1qY9z3mN4pL8vO...

boolean valid = checkPassword("user123", hashed); // true
````

Add the dependency:

````xml
<dependency>
    <groupId>org.mindrot</groupId>
    <artifactId>jbcrypt</artifactId>
    <version>0.4</version>
</dependency>
````

**ProTip**: Use bcrypt’s `gensalt()` with a work factor (e.g., 12) to balance security and performance for password
hashing.

## Common Pitfalls and Best Practices

Hashing is powerful, but I’ve tripped up:

- **Using Weak Algorithms**: Early in Co-op’s system, I used MD5 for sensitive data—big mistake. Stick to SHA-256 or
  bcrypt for security.
- **Ignoring Encoding**: In Mosaic’s pipeline, inconsistent UTF-8 encoding broke hash comparisons. Always specify
  `StandardCharsets.UTF_8`.
- **Not Handling Exceptions**: In Ribby Hall’s sync, I forgot to catch `NoSuchAlgorithmException`, crashing the app.
  Always wrap hashing in try-catch.
- **Overusing bcrypt**: In ESG’s system, I tried bcrypt for non-password data, slowing things down. Use fast algorithms
  like SHA-256 for non-passwords.

**ProTip**: Profile hashing performance with VisualVM in high-throughput systems like Mosaic’s pipeline to catch
bottlenecks.

## Conclusion

Hashing has been a cornerstone of my Java projects. In ESG’s BOL Engine, bcrypt kept user passwords secure. In Mosaic’s
pipeline, SHA-256 ensured trade event integrity. In Co-op’s reports and Ribby Hall’s sync, hashing sped up lookups and
verified data. Whether you’re securing credentials or optimizing lookups, Java’s hashing tools have you covered. Start
small: try SHA-256 for a checksum or bcrypt for a password.

Got a hashing trick that saved your day? Ping me [here]({{ site.baseurl }}/contact/), I’d love to hear your story!

<p class="text-center">
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/security/MessageDigest.html" text="MessageDigest" %}
</p>