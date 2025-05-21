---
title: Apache HttpClient Classic APIs | A Quick Guide
tags: [ Java, HttpClient, Apache, APIs, HTTP ]
style: fill
color: primary
description: As a Java developer, I’ve used Apache HttpClient Classic APIs to handle HTTP requests. Here’s my guide to mastering HTTP requests with Apache HttpClient Classic APIs, with real-world tips.
---

---

# Apache HttpClient Classic APIs: A Quick Guide

Apache HttpClient's classic APIs make synchronous HTTP requests in Java simple and powerful. This guide shows how to use
them for CRUD operations with the [Reqres API](https://reqres.in), a free mock API for testing. We'll cover POST, GET,
HEAD, OPTIONS, PUT, PATCH, and DELETE with code and tests.

## Why Apache HttpClient?

The classic APIs are synchronous, ideal for straightforward HTTP tasks. They support everything from basic GETs to
complex POSTs, with response handlers for clean, reusable code.

## Setup

Use the Reqres API (`https://reqres.in`) for testing. It supports all HTTP methods for CRUD on `/api/users`.

## CRUD Operations

- **Create**: POST `/api/users`
- **Read**: GET `/api/users/{id}` or `/api/users?page={n}`
- **Update**: PUT or PATCH `/api/users/{id}`
- **Delete**: DELETE `/api/users/{id}`

Response handlers streamline response processing, error handling, and resource cleanup.

## HTTP Methods

### POST: Create a User

Send a POST request to create a user.

```java
public String createUser(String firstName, String lastName, String email, String avatar)
        throws RequestProcessingException {
    try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
        List<NameValuePair> formParams = new ArrayList<>();
        formParams.add(new BasicNameValuePair("first_name", firstName));
        formParams.add(new BasicNameValuePair("last_name", lastName));
        formParams.add(new BasicNameValuePair("email", email));
        formParams.add(new BasicNameValuePair("avatar", avatar));
        try (UrlEncodedFormEntity entity = new UrlEncodedFormEntity(formParams, StandardCharsets.UTF_8)) {
            HttpHost httpHost = HttpHost.create("https://reqres.in");
            URI uri = new URIBuilder("/api/users/").build();
            HttpPost httpPostRequest = new HttpPost(uri);
            httpPostRequest.setEntity(entity);
            BasicHttpClientResponseHandler handler = new BasicHttpClientResponseHandler();
            return httpClient.execute(httpHost, httpPostRequest, handler);
        }
    } catch (Exception e) {
        throw new RequestProcessingException("Failed to create user.", e);
    }
}
```

**Test**:

```java

@Test
void executePostRequest() {
    try {
        String createdUser = userHttpRequestHelper.createUser(
                "DummyFirst", "DummyLast", "DummyEmail@example.com", "DummyAvatar");
        assertThat(createdUser).isNotEmpty();
    } catch (Exception e) {
        Assertions.fail("Failed to execute HTTP request.", e);
    }
}
```

### GET: Read Users

Fetch a single user or paginated list.

**Paginated GET**:

```java
public String getPaginatedUsers(Map<String, String> requestParameters)
        throws RequestProcessingException {
    try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
        HttpHost httpHost = HttpHost.create("https://reqres.in");
        List<NameValuePair> nameValuePairs = requestParameters.entrySet().stream()
                .map(entry -> new BasicNameValuePair(entry.getKey(), entry.getValue()))
                .toList();
        URI uri = new URIBuilder("/api/users/").addParameters(nameValuePairs).build();
        HttpGet httpGetRequest = new HttpGet(uri);
        BasicHttpClientResponseHandler handler = new BasicHttpClientResponseHandler();
        return httpClient.execute(httpHost, httpGetRequest, handler);
    } catch (Exception e) {
        throw new RequestProcessingException("Failed to get paginated users.", e);
    }
}
```

**Test**:

```java

@Test
void executeGetPaginatedRequest() {
    try {
        Map<String, String> params = Map.of("page", "1");
        String responseBody = userHttpRequestHelper.getPaginatedUsers(params);
        assertThat(responseBody).isNotEmpty();
    } catch (Exception e) {
        Assertions.fail("Failed to execute HTTP request.", e);
    }
}
```

**Specific User GET**:

```java
public String getUser(long userId) throws RequestProcessingException {
    try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
        HttpHost httpHost = HttpHost.create("https://reqres.in");
        HttpGet httpGetRequest = new HttpGet(new URIBuilder("/api/users/" + userId).build());
        BasicHttpClientResponseHandler handler = new BasicHttpClientResponseHandler();
        return httpClient.execute(httpHost, httpGetRequest, handler);
    } catch (Exception e) {
        throw new RequestProcessingException(
                MessageFormat.format("Failed to get user for ID: {0}", userId), e);
    }
}
```

**Test**:

```java

@Test
void executeGetSpecificRequest() {
    try {
        long userId = 2L;
        String existingUser = userHttpRequestHelper.getUser(userId);
        assertThat(existingUser).isNotEmpty();
    } catch (Exception e) {
        Assertions.fail("Failed to execute HTTP request.", e);
    }
}
```

### HEAD: Check Status

Verify a resource’s status without fetching its body.

```java
public Integer getUserStatus(long userId) throws RequestProcessingException {
    try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
        HttpHost httpHost = HttpHost.create("https://reqres.in");
        URI uri = new URIBuilder("/api/users/" + userId).build();
        HttpHead httpHeadRequest = new HttpHead(uri);
        HttpClientResponseHandler<Integer> handler = HttpResponse::getCode;
        Integer code = httpClient.execute(httpHost, httpHeadRequest, handler);
        return code;
    } catch (Exception e) {
        throw new RequestProcessingException(
                MessageFormat.format("Failed to get user for ID: {0}", userId), e);
    }
}
```

**Test**:

```java

@Test
void executeUserStatus() {
    try {
        long userId = 2L;
        Integer userStatus = userHttpRequestHelper.getUserStatus(userId);
        assertThat(userStatus).isEqualTo(HttpStatus.SC_OK);
    } catch (Exception e) {
        Assertions.fail("Failed to execute HTTP request.", e);
    }
}
```

### OPTIONS: Discover Methods

Check which HTTP methods the server supports.

```java
public Map<String, String> executeOptions() throws RequestProcessingException {
    try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
        HttpHost httpHost = HttpHost.create("https://reqres.in");
        URI uri = new URIBuilder("/api/users/").build();
        HttpOptions httpOptionsRequest = new HttpOptions(uri);
        HttpClientResponseHandler<Map<String, String>> handler = response ->
                StreamSupport.stream(
                                Spliterators.spliteratorUnknownSize(response.headerIterator(), Spliterator.ORDERED), false)
                        .collect(Collectors.toMap(Header::getName, Header::getValue));
        return httpClient.execute(httpHost, httpOptionsRequest, handler);
    } catch (Exception e) {
        throw new RequestProcessingException("Failed to execute the request.", e);
    }
}
```

**Test**:

```java

@Test
void executeOptions() {
    try {
        Map<String, String> headers = userHttpRequestHelper.executeOptions();
        assertThat(headers.keySet())
                .as("Headers do not contain allow header")
                .containsAnyOf("Allow", "Access-Control-Allow-Methods");
    } catch (Exception e) {
        Assertions.fail("Failed to execute HTTP request.", e);
    }
}
```

### PUT: Update a User

Update an entire user record.

```java
public String updateUser(long userId, String firstName, String lastName, String email, String avatar)
        throws RequestProcessingException {
    try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
        List<NameValuePair> formParams = new ArrayList<>();
        formParams.add(new BasicNameValuePair("first_name", firstName));
        formParams.add(new BasicNameValuePair("last_name", lastName));
        formParams.add(new BasicNameValuePair("email", email));
        formParams.add(new BasicNameValuePair("avatar", avatar));
        try (UrlEncodedFormEntity entity = new UrlEncodedFormEntity(formParams, StandardCharsets.UTF_8)) {
            HttpHost httpHost = HttpHost.create("https://reqres.in");
            URI uri = new URIBuilder("/api/users/" + userId).build();
            HttpPut httpPutRequest = new HttpPut(uri);
            httpPutRequest.setEntity(entity);
            BasicHttpClientResponseHandler handler = new BasicHttpClientResponseHandler();
            return httpClient.execute(httpHost, httpPutRequest, handler);
        }
    } catch (Exception e) {
        throw new RequestProcessingException("Failed to update user.", e);
    }
}
```

**Test**:

```java

@Test
void executePutRequest() {
    try {
        int userId = 2;
        String updatedUser = userHttpRequestHelper.updateUser(
                userId, "UpdatedDummyFirst", "UpdatedDummyLast", "UpdatedDummyEmail@example.com", "UpdatedDummyAvatar");
        assertThat(updatedUser).isNotEmpty();
    } catch (Exception e) {
        Assertions.fail("Failed to execute HTTP request.", e);
    }
}
```

### PATCH: Partial Update

Update specific user fields.

```java
public String patchUser(long userId, String firstName, String lastName)
        throws RequestProcessingException {
    try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
        List<NameValuePair> formParams = new ArrayList<>();
        formParams.add(new BasicNameValuePair("first_name", firstName));
        formParams.add(new BasicNameValuePair("last_name", lastName));
        try (UrlEncodedFormEntity entity = new UrlEncodedFormEntity(formParams, StandardCharsets.UTF_8)) {
            HttpHost httpHost = HttpHost.create("https://reqres.in");
            URI uri = new URIBuilder("/api/users/" + userId).build();
            HttpPatch httpPatchRequest = new HttpPatch(uri);
            httpPatchRequest.setEntity(entity);
            BasicHttpClientResponseHandler handler = new BasicHttpClientResponseHandler();
            return httpClient.execute(httpHost, httpPatchRequest, handler);
        }
    } catch (Exception e) {
        throw new RequestProcessingException("Failed to patch user.", e);
    }
}
```

**Test**:

```java

@Test
void executePatchRequest() {
    try {
        int userId = 2;
        String patchedUser = userHttpRequestHelper.patchUser(userId, "UpdatedDummyFirst", "UpdatedDummyLast");
        assertThat(patchedUser).isNotEmpty();
    } catch (Exception e) {
        Assertions.fail("Failed to execute HTTP request.", e);
    }
}
```

### DELETE: Remove a User

Delete a user by ID.

```java
public void deleteUser(long userId) throws RequestProcessingException {
    try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
        HttpHost httpHost = HttpHost.create("https://reqres.in");
        URI uri = new URIBuilder("/api/users/" + userId).build();
        HttpDelete httpDeleteRequest = new HttpDelete(uri);
        BasicHttpClientResponseHandler handler = new BasicHttpClientResponseHandler();
        httpClient.execute(httpHost, httpDeleteRequest, handler);
    } catch (Exception e) {
        throw new RequestProcessingException("Failed to delete user.", e);
    }
}
```

**Test**:

```java

@Test
void executeDeleteRequest() {
    try {
        int userId = 2;
        userHttpRequestHelper.deleteUser(userId);
    } catch (Exception e) {
        Assertions.fail("Failed to execute HTTP request.", e);
    }
}
```

## Custom Types

Use POJOs for type-safe requests and responses. Serialize to JSON with Jackson for HTTP entities.

```java
public class DataObjectResponseHandler<T> extends AbstractHttpClientResponseHandler<T> {
    private ObjectMapper objectMapper = new ObjectMapper();
    @NonNull
    private Class<T> realType;

    public DataObjectResponseHandler(@NonNull Class<T> realType) {
        this.realType = realType;
    }

    @Override
    public T handleEntity(HttpEntity httpEntity) throws IOException {
        try {
            return objectMapper.readValue(EntityUtils.toString(httpEntity), realType);
        } catch (ParseException e) {
            throw new ClientProtocolException(e);
        }
    }
}

public class UserTypeHttpRequestHelper extends BaseHttpRequestHelper {
    public User getUser(long userId) throws RequestProcessingException {
        try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
            HttpHost httpHost = userRequestProcessingUtils.getApiHost();
            URI uri = userRequestProcessingUtils.prepareUsersApiUri(userId);
            HttpGet httpGetRequest = new HttpGet(uri);
            HttpClientResponseHandler<User> handler = new DataObjectResponseHandler<>(User.class);
            return httpClient.execute(httpHost, httpGetRequest, handler);
        } catch (Exception e) {
            throw new RequestProcessingException(
                    MessageFormat.format("Failed to get user for ID: {0}", userId), e);
        }
    }
}
```

**Test**:

```java

@Test
void executeGetUser() {
    try {
        long userId = 2L;
        User existingUser = userHttpRequestHelper.getUser(userId);
        ThrowingConsumer<User> responseRequirements = user -> {
            assertThat(user).as("Created user cannot be null.").isNotNull();
            assertThat(user.getId()).as("ID should be positive number.").isEqualTo(userId);
            assertThat(user.getFirstName()).as("First name cannot be null.").isNotEmpty();
            assertThat(user.getLastName()).as("Last name cannot be null.").isNotEmpty();
            assertThat(user.getAvatar()).as("Avatar cannot be null.").isNotNull();
        };
        assertThat(existingUser).satisfies(responseRequirements);
    } catch (Exception e) {
        Assertions.fail("Failed to execute HTTP request.", e);
    }
}
```

POJOs enhance type safety but add complexity compared to strings.

## Conclusion

Apache HttpClient’s classic APIs simplify synchronous HTTP requests for CRUD operations. With response handlers and
custom types, you can build robust, maintainable Java applications. Test these examples with Reqres and explore further!


<p class="text-center">
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://hc.apache.org/httpcomponents-client-4.5.x/index.html" text="Apache HttpClient" %}
{% include elements/button.html link="https://reqres.in/" text="Reqres API" %}
</p>