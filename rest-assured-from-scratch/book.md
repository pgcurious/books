# Inventing Rest Assured from First Principles
## The Journey from Manual HTTP Testing to Elegant API Verification

---

## Introduction: A World Without Rest Assured

Imagine you're a backend engineer in 2010. You've just built a Spring Boot REST API:

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

Your boss walks in: "Great work! Now... how do we know it actually works?"

You stare at your code. You need to test it. But how?

**This book is about solving that problem.** Not by memorizing Rest Assured's syntax, but by *inventing* it ourselves, one painful problem at a time.

By the end, you won't just know **how** to use Rest Assured. You'll understand **why it exists** and why every design decision was made.

Let's start with nothing and build everything.

---

## Chapter 1: The Testing Crisis

### The Problem

You have a REST API. You need to verify:
- Does it return the right HTTP status code?
- Is the JSON response correct?
- Are the headers set properly?
- Does error handling work?

### Attempt 1: Manual Testing with cURL

**The naive approach**:

```bash
# Test GET endpoint
curl http://localhost:8080/api/users/1

# Response:
# {"id":1,"name":"Alice","email":"alice@example.com"}

# Did it work? ¯\_(ツ)_/¯
# You manually check:
# ✓ Status 200? (I think so...)
# ✓ JSON looks right? (Seems legit...)
# ✓ Email field present? (Yep!)
```

**The Problems**:
1. **Not automated** - You have to run it manually every time
2. **No assertions** - You visually verify correctness
3. **Not repeatable** - Hard to run 100 endpoints this way
4. **No CI/CD integration** - Can't run in build pipeline
5. **Error-prone** - Easy to miss bugs

### Attempt 2: Manual Testing with Postman

Better! You can:
- Save requests
- Create collections
- Write some tests

```javascript
// Postman test
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("User has email", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.email).to.exist;
});
```

**Still problematic**:
1. **Outside your codebase** - Tests live in Postman, not Git
2. **Manual execution** - Have to click "Run Collection"
3. **No IDE integration** - Can't debug tests in IntelliJ
4. **Team sync issues** - How do you share collections?
5. **Not in CI/CD** - Can't easily run in Jenkins/GitHub Actions

### The Insight

**We need programmatic API testing** - Tests written in Java, living in our codebase, running automatically.

**Question**: How do we test HTTP APIs from Java code?

---

## Chapter 2: Attempt 1 — Pure Java HttpURLConnection

### The Simplest Solution

Java has built-in HTTP support! Let's use it:

```java
@Test
public void testGetUser() throws Exception {
    // Create connection
    URL url = new URL("http://localhost:8080/api/users/1");
    HttpURLConnection connection = (HttpURLConnection) url.openConnection();
    connection.setRequestMethod("GET");

    // Get response code
    int responseCode = connection.getResponseCode();

    // Read response body
    BufferedReader reader = new BufferedReader(
        new InputStreamReader(connection.getInputStream())
    );
    StringBuilder response = new StringBuilder();
    String line;
    while ((line = reader.readLine()) != null) {
        response.append(line);
    }
    reader.close();

    // Parse JSON manually
    String jsonResponse = response.toString();

    // Assert (somehow?)
    assertEquals(200, responseCode);
    assertTrue(jsonResponse.contains("\"email\""));
    assertTrue(jsonResponse.contains("alice@example.com"));
}
```

### The Problems

**Way too verbose**:
- 20+ lines just to make one GET request
- Manual JSON parsing with string manipulation
- No structured assertions
- Connection management is tedious

**Testing a POST is even worse**:

```java
@Test
public void testCreateUser() throws Exception {
    URL url = new URL("http://localhost:8080/api/users");
    HttpURLConnection connection = (HttpURLConnection) url.openConnection();
    connection.setRequestMethod("POST");
    connection.setRequestProperty("Content-Type", "application/json");
    connection.setDoOutput(true);

    // Write request body
    String jsonInput = "{\"name\":\"Bob\",\"email\":\"bob@example.com\"}";
    OutputStream os = connection.getOutputStream();
    byte[] input = jsonInput.getBytes("utf-8");
    os.write(input, 0, input.length);
    os.close();

    // Read response
    BufferedReader reader = new BufferedReader(
        new InputStreamReader(connection.getInputStream())
    );
    StringBuilder response = new StringBuilder();
    String line;
    while ((line = reader.readLine()) != null) {
        response.append(line);
    }
    reader.close();

    // Assert
    int responseCode = connection.getResponseCode();
    assertEquals(201, responseCode);
    assertTrue(response.toString().contains("\"name\":\"Bob\""));
}
```

**30+ lines for a simple POST test!**

### The Insight

**We need an abstraction** - Something that hides the HTTP plumbing and lets us focus on **what** we're testing, not **how** HTTP works.

---

## Chapter 3: Attempt 2 — Spring's RestTemplate

### The Idea

Spring provides `RestTemplate` - a higher-level HTTP client:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class UserApiTest {

    @LocalServerPort
    private int port;

    private RestTemplate restTemplate = new RestTemplate();

    @Test
    public void testGetUser() {
        String url = "http://localhost:" + port + "/api/users/1";

        ResponseEntity<User> response = restTemplate.getForEntity(url, User.class);

        assertEquals(HttpStatus.OK, response.getStatusCode());
        assertNotNull(response.getBody());
        assertEquals("alice@example.com", response.getBody().getEmail());
    }
}
```

**Much better!**

- No manual connection management
- Automatic JSON deserialization to `User` object
- Cleaner assertions

### Testing POST:

```java
@Test
public void testCreateUser() {
    String url = "http://localhost:" + port + "/api/users";

    User newUser = new User();
    newUser.setName("Bob");
    newUser.setEmail("bob@example.com");

    ResponseEntity<User> response = restTemplate.postForEntity(url, newUser, User.class);

    assertEquals(HttpStatus.CREATED, response.getStatusCode());
    assertNotNull(response.getBody());
    assertEquals("Bob", response.getBody().getName());
}
```

**Pretty good!** Down to ~10 lines.

### But... The Problems Emerge

**Problem 1: Complex JSON assertions**

```java
// What if response is:
// {
//   "user": {
//     "id": 1,
//     "name": "Alice",
//     "addresses": [
//       {"city": "NYC", "zip": "10001"},
//       {"city": "LA", "zip": "90001"}
//     ]
//   },
//   "metadata": {
//     "timestamp": "2024-11-10T12:00:00Z"
//   }
// }

// You need a whole class hierarchy!
ResponseEntity<UserResponse> response = restTemplate.getForEntity(url, UserResponse.class);

// Or parse as Map:
ResponseEntity<Map> response = restTemplate.getForEntity(url, Map.class);
Map<String, Object> body = response.getBody();
Map<String, Object> user = (Map<String, Object>) body.get("user");
List<Map<String, Object>> addresses = (List<Map<String, Object>>) user.get("addresses");
String zip = (String) addresses.get(0).get("zip");

assertEquals("10001", zip);  // SO MUCH BOILERPLATE!
```

**Problem 2: Headers are tedious**

```java
@Test
public void testWithAuthHeader() {
    HttpHeaders headers = new HttpHeaders();
    headers.set("Authorization", "Bearer token123");
    headers.setContentType(MediaType.APPLICATION_JSON);

    HttpEntity<User> request = new HttpEntity<>(newUser, headers);

    ResponseEntity<User> response = restTemplate.exchange(
        url,
        HttpMethod.POST,
        request,
        User.class
    );

    // ...
}
```

**Problem 3: Query parameters are painful**

```java
@Test
public void testSearchUsers() {
    String url = "http://localhost:" + port + "/api/users?name=Alice&status=active&limit=10";

    // Or with UriComponentsBuilder:
    String url = UriComponentsBuilder
        .fromHttpUrl("http://localhost:" + port + "/api/users")
        .queryParam("name", "Alice")
        .queryParam("status", "active")
        .queryParam("limit", 10)
        .toUriString();

    // Still verbose!
}
```

**Problem 4: Response validation is scattered**

```java
@Test
public void testCompleteValidation() {
    // Check status
    assertEquals(HttpStatus.OK, response.getStatusCode());

    // Check header
    assertEquals("application/json", response.getHeaders().getContentType().toString());

    // Check body
    User user = response.getBody();
    assertNotNull(user);
    assertEquals("Alice", user.getName());
    assertEquals("alice@example.com", user.getEmail());

    // Check nested field
    assertNotNull(user.getAddresses());
    assertEquals(2, user.getAddresses().size());
    assertEquals("NYC", user.getAddresses().get(0).getCity());
}

// Tests are hard to read - what's the important part?
```

### The Insight

**We need**:
1. **Fluent API** - Chain method calls for readability
2. **Flexible JSON validation** - Don't require full POJO classes
3. **Built-in assertions** - Status, headers, body in one place
4. **Readable syntax** - Tests should read like documentation

**This is where Rest Assured comes in. Let's invent it!**

---

## Chapter 4: Inventing the Fluent API

### The Vision

What if our test could read like this:

```java
@Test
public void testGetUser() {
    given()
        .when()
            .get("/api/users/1")
        .then()
            .statusCode(200)
            .body("email", equalTo("alice@example.com"));
}
```

**Beautiful!** Reads like a sentence:
- **Given** (preconditions)
- **When** (action)
- **Then** (assertions)

This is **Behavior-Driven Development (BDD)** style!

### Building It: Step 1 — RequestSpecification

Let's build the `given()` part:

```java
public class RequestSpecification {
    private Map<String, String> headers = new HashMap<>();
    private Map<String, Object> queryParams = new HashMap<>();
    private Object body;
    private String baseUri = "http://localhost:8080";

    public RequestSpecification header(String name, String value) {
        headers.put(name, value);
        return this;  // Return 'this' for chaining!
    }

    public RequestSpecification queryParam(String name, Object value) {
        queryParams.put(name, value);
        return this;
    }

    public RequestSpecification body(Object body) {
        this.body = body;
        return this;
    }

    public RequestSpecification baseUri(String baseUri) {
        this.baseUri = baseUri;
        return this;
    }

    public Response get(String path) {
        // Execute GET request with accumulated configuration
        return executeRequest("GET", path);
    }

    public Response post(String path) {
        return executeRequest("POST", path);
    }

    private Response executeRequest(String method, String path) {
        // Build URL with query params
        // Set headers
        // Send body if present
        // Return Response object
    }
}
```

### Building It: Step 2 — Response Validation

```java
public class Response {
    private int statusCode;
    private String body;
    private Map<String, String> headers;

    public Response statusCode(int expected) {
        if (this.statusCode != expected) {
            throw new AssertionError(
                "Expected status code " + expected + " but was " + statusCode
            );
        }
        return this;
    }

    public Response header(String name, String expectedValue) {
        String actual = headers.get(name);
        if (!expectedValue.equals(actual)) {
            throw new AssertionError(
                "Expected header '" + name + "' to be '" + expectedValue +
                "' but was '" + actual + "'"
            );
        }
        return this;
    }

    public Response body(String jsonPath, Object expectedValue) {
        // Extract value from JSON using path
        Object actual = extractFromJson(jsonPath);
        if (!expectedValue.equals(actual)) {
            throw new AssertionError(
                "Expected body path '" + jsonPath + "' to be '" + expectedValue +
                "' but was '" + actual + "'"
            );
        }
        return this;
    }
}
```

### Building It: Step 3 — The Given-When-Then DSL

```java
public class RestAssured {

    public static RequestSpecification given() {
        return new RequestSpecification();
    }
}

// Usage:
import static RestAssured.*;

@Test
public void test() {
    given()
        .header("Authorization", "Bearer token")
        .queryParam("status", "active")
        .when()
            .get("/api/users")
        .then()
            .statusCode(200);
}
```

**Wait, what's `when()`?**

It's just a readability method:

```java
public class RequestSpecification {
    // ...

    public RequestSpecification when() {
        return this;  // Does nothing, just for readability!
    }
}
```

**What about `then()`?**

```java
public class Response {
    // Already returns 'this' from assertion methods
    // So then() is implicit!
}

// But we can make it explicit:
public class Response {
    public Response then() {
        return this;  // Again, just for readability
    }
}
```

### The Pattern: Fluent Interface

**The secret**: Every method returns `this` (or the next object in the chain).

```java
given()           // Returns RequestSpecification
  .header(...)    // Returns RequestSpecification
  .queryParam(...)// Returns RequestSpecification
  .when()         // Returns RequestSpecification
  .get("/path")   // Returns Response
  .then()         // Returns Response
  .statusCode(...)// Returns Response
  .body(...)      // Returns Response
```

**This is called the Fluent Interface pattern!**

---

## Chapter 5: The JSON Path Problem

### The Challenge

Your API returns:

```json
{
  "user": {
    "id": 1,
    "name": "Alice",
    "emails": ["alice@work.com", "alice@home.com"],
    "address": {
      "city": "NYC",
      "zip": "10001"
    }
  },
  "metadata": {
    "timestamp": "2024-11-10T12:00:00Z",
    "version": "2.0"
  }
}
```

How do you assert `user.address.city` equals "NYC"?

### Attempt 1: Parse to POJO

```java
@Test
public void test() {
    ResponseEntity<UserResponse> response = restTemplate.getForEntity(url, UserResponse.class);

    UserResponse body = response.getBody();
    assertEquals("NYC", body.getUser().getAddress().getCity());
}

// Requires:
class UserResponse {
    private User user;
    private Metadata metadata;
    // getters/setters...
}

class User {
    private Long id;
    private String name;
    private List<String> emails;
    private Address address;
    // getters/setters...
}

class Address {
    private String city;
    private String zip;
    // getters/setters...
}

class Metadata {
    private String timestamp;
    private String version;
    // getters/setters...
}

// SO MUCH BOILERPLATE for one assertion!
```

### Attempt 2: Parse to Map

```java
@Test
public void test() {
    ResponseEntity<Map> response = restTemplate.getForEntity(url, Map.class);

    Map<String, Object> body = response.getBody();
    Map<String, Object> user = (Map<String, Object>) body.get("user");
    Map<String, Object> address = (Map<String, Object>) user.get("address");
    String city = (String) address.get("city");

    assertEquals("NYC", city);
}

// Better, but lots of casting and null checks needed!
```

### The Solution: JSON Path

**Idea**: Use a path syntax to navigate JSON:

```java
String city = JsonPath.read(json, "user.address.city");
assertEquals("NYC", city);
```

**Even better with Rest Assured**:

```java
given()
    .when()
        .get("/api/users/1")
    .then()
        .body("user.address.city", equalTo("NYC"))
        .body("user.emails[0]", equalTo("alice@work.com"))
        .body("metadata.version", equalTo("2.0"));
```

**No POJOs needed!**

### Implementing JSON Path Support

```java
public class Response {
    private String jsonBody;

    public Response body(String jsonPath, Matcher<?> matcher) {
        // Parse JSON
        Object value = extractJsonPath(jsonPath);

        // Use Hamcrest matcher
        if (!matcher.matches(value)) {
            throw new AssertionError(
                "JSON path '" + jsonPath + "' " + matcher.toString()
            );
        }

        return this;
    }

    private Object extractJsonPath(String path) {
        // Parse path: "user.address.city" -> ["user", "address", "city"]
        String[] parts = path.split("\\.");

        Object current = parseJson(jsonBody);  // Parse to Map/List

        for (String part : parts) {
            if (part.contains("[")) {
                // Array access: "emails[0]"
                String key = part.substring(0, part.indexOf("["));
                int index = Integer.parseInt(
                    part.substring(part.indexOf("[") + 1, part.indexOf("]"))
                );
                current = ((Map) current).get(key);
                current = ((List) current).get(index);
            } else {
                // Object access: "user"
                current = ((Map) current).get(part);
            }
        }

        return current;
    }
}
```

### The Real JSON Path Library

In reality, Rest Assured uses **Groovy's GPath** and the **JsonPath library**:

```java
// Simple paths
.body("name", equalTo("Alice"))

// Nested paths
.body("user.address.city", equalTo("NYC"))

// Array access
.body("emails[0]", equalTo("alice@work.com"))

// Array size
.body("emails.size()", equalTo(2))

// Find in array
.body("users.find { it.name == 'Alice' }.email", equalTo("alice@work.com"))

// Collect from array
.body("users.collect { it.name }", hasItems("Alice", "Bob"))
```

**Powerful and concise!**

---

## Chapter 6: The Spring Boot Integration

### The Problem

You're testing a Spring Boot app. You need:
1. Start the server
2. Run tests against it
3. Shut down the server

### Attempt 1: Manual Server Start

```java
public class UserApiTest {
    private static ConfigurableApplicationContext context;

    @BeforeAll
    public static void setUp() {
        SpringApplication app = new SpringApplication(Application.class);
        context = app.run();
    }

    @AfterAll
    public static void tearDown() {
        context.close();
    }

    @Test
    public void test() {
        given()
            .baseUri("http://localhost:8080")
            .when()
                .get("/api/users/1")
            .then()
                .statusCode(200);
    }
}
```

**Problems**:
- Hard-coded port (what if 8080 is in use?)
- Manual lifecycle management
- No Spring test context integration

### The Solution: @SpringBootTest

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class UserApiTest {

    @LocalServerPort
    private int port;

    @BeforeEach
    public void setUp() {
        RestAssured.port = port;
    }

    @Test
    public void testGetUser() {
        given()
            .when()
                .get("/api/users/1")
            .then()
                .statusCode(200)
                .body("name", equalTo("Alice"));
    }
}
```

**Features**:
- ✓ Random port (no conflicts!)
- ✓ Full Spring context loaded
- ✓ All beans available
- ✓ Database integration works
- ✓ Clean test isolation

### Better: Extract Base URI Setup

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class UserApiTest {

    @LocalServerPort
    private int port;

    @BeforeEach
    public void setUp() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
        RestAssured.basePath = "/api";
    }

    @Test
    public void testGetUser() {
        given()
            .when()
                .get("/users/1")  // Cleaner!
            .then()
                .statusCode(200);
    }
}
```

### Even Better: Base Test Class

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class ApiTestBase {

    @LocalServerPort
    private int port;

    @BeforeEach
    public void setUpRestAssured() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
        RestAssured.basePath = "/api";
    }

    @AfterEach
    public void resetRestAssured() {
        RestAssured.reset();  // Clean up for next test
    }
}

// Now all tests are clean:
public class UserApiTest extends ApiTestBase {

    @Test
    public void testGetUser() {
        given()
            .when()
                .get("/users/1")
            .then()
                .statusCode(200)
                .body("name", equalTo("Alice"));
    }
}
```

**Clean and reusable!**

---

## Chapter 7: The Authentication Problem

### The Challenge

Most APIs require authentication:

```java
@Test
public void testProtectedEndpoint() {
    given()
        .header("Authorization", "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...")
        .when()
            .get("/users/me")
        .then()
            .statusCode(200);
}
```

**Problems**:
1. Token is hard-coded
2. Every test needs to add the header
3. Token might expire
4. No way to get a fresh token

### Solution 1: Extract Token Generation

```java
public abstract class ApiTestBase {

    protected String getAuthToken() {
        return given()
            .contentType("application/json")
            .body(new LoginRequest("test@example.com", "password"))
            .when()
                .post("/auth/login")
            .then()
                .statusCode(200)
                .extract()
                .path("token");
    }
}

@Test
public void testProtectedEndpoint() {
    String token = getAuthToken();

    given()
        .header("Authorization", "Bearer " + token)
        .when()
            .get("/users/me")
        .then()
            .statusCode(200);
}
```

**Better, but still repetitive!**

### Solution 2: RequestSpecification Reuse

```java
public abstract class ApiTestBase {

    protected RequestSpecification authenticatedRequest() {
        String token = getAuthToken();

        return given()
            .header("Authorization", "Bearer " + token);
    }
}

@Test
public void testProtectedEndpoint() {
    authenticatedRequest()
        .when()
            .get("/users/me")
        .then()
            .statusCode(200);
}
```

**Much cleaner!**

### Solution 3: Rest Assured's RequestSpecBuilder

```java
public abstract class ApiTestBase {

    @LocalServerPort
    private int port;

    protected RequestSpecification authenticatedSpec;

    @BeforeEach
    public void setUp() {
        RestAssured.port = port;

        String token = getAuthToken();

        authenticatedSpec = new RequestSpecBuilder()
            .setBaseUri("http://localhost")
            .setPort(port)
            .setBasePath("/api")
            .addHeader("Authorization", "Bearer " + token)
            .setContentType(ContentType.JSON)
            .build();
    }
}

@Test
public void testProtectedEndpoint() {
    given()
        .spec(authenticatedSpec)  // Reuse the spec!
        .when()
            .get("/users/me")
        .then()
            .statusCode(200);
}
```

### Solution 4: OAuth2 Helper

```java
public abstract class ApiTestBase {

    protected RequestSpecification authenticatedSpec;

    @BeforeEach
    public void setUp() {
        authenticatedSpec = new RequestSpecBuilder()
            .setBaseUri("http://localhost")
            .setPort(port)
            .setBasePath("/api")
            .setAuth(oauth2(getAccessToken()))  // Rest Assured's OAuth2 support!
            .build();
    }

    private String getAccessToken() {
        return given()
            .formParam("grant_type", "client_credentials")
            .formParam("client_id", "test-client")
            .formParam("client_secret", "test-secret")
            .when()
                .post("/oauth/token")
            .then()
                .extract()
                .path("access_token");
    }
}
```

**Rest Assured has built-in OAuth2 support!**

---

## Chapter 8: The Test Data Problem

### The Challenge

Tests need data in the database:

```java
@Test
public void testGetUser() {
    // Problem: User with ID 1 might not exist!
    given()
        .when()
            .get("/users/1")
        .then()
            .statusCode(200);
}
```

### Solution 1: Test Data Setup

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class UserApiTest extends ApiTestBase {

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    public void setUpData() {
        userRepository.deleteAll();

        User alice = new User();
        alice.setName("Alice");
        alice.setEmail("alice@example.com");
        userRepository.save(alice);
    }

    @Test
    public void testGetUser() {
        given()
            .when()
                .get("/users/1")
            .then()
                .statusCode(200)
                .body("name", equalTo("Alice"));
    }
}
```

**Better: Extract to helper methods**

```java
public abstract class ApiTestBase {

    @Autowired
    protected UserRepository userRepository;

    protected User createUser(String name, String email) {
        User user = new User();
        user.setName(name);
        user.setEmail(email);
        return userRepository.save(user);
    }

    protected void cleanDatabase() {
        userRepository.deleteAll();
    }
}

@Test
public void testGetUser() {
    User alice = createUser("Alice", "alice@example.com");

    given()
        .when()
            .get("/users/" + alice.getId())
        .then()
            .statusCode(200)
            .body("name", equalTo("Alice"));
}
```

### Solution 2: Test Data Builders

```java
public class UserBuilder {
    private String name = "Default Name";
    private String email = "default@example.com";
    private String status = "ACTIVE";

    public UserBuilder withName(String name) {
        this.name = name;
        return this;
    }

    public UserBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public UserBuilder inactive() {
        this.status = "INACTIVE";
        return this;
    }

    public User build() {
        User user = new User();
        user.setName(name);
        user.setEmail(email);
        user.setStatus(status);
        return user;
    }

    public User buildAndSave(UserRepository repository) {
        return repository.save(build());
    }
}

@Test
public void testGetInactiveUser() {
    User bob = new UserBuilder()
        .withName("Bob")
        .withEmail("bob@example.com")
        .inactive()
        .buildAndSave(userRepository);

    given()
        .when()
            .get("/users/" + bob.getId())
        .then()
            .statusCode(200)
            .body("status", equalTo("INACTIVE"));
}
```

**Fluent and flexible!**

---

## Chapter 9: The Response Extraction Problem

### The Challenge

Sometimes you need data from a response to use in the next request:

```java
@Test
public void testCreateThenUpdate() {
    // Create user
    // How do we get the ID from the response?

    // Update user
    // Need to use the ID from the previous step
}
```

### Solution: Extract Response Values

```java
@Test
public void testCreateThenUpdate() {
    // Create user and extract ID
    Long userId = given()
        .contentType("application/json")
        .body(new User("Alice", "alice@example.com"))
        .when()
            .post("/users")
        .then()
            .statusCode(201)
            .extract()
            .path("id");  // Extract the ID!

    // Update user using extracted ID
    given()
        .contentType("application/json")
        .body(Map.of("name", "Alice Updated"))
        .when()
            .put("/users/" + userId)
        .then()
            .statusCode(200)
            .body("name", equalTo("Alice Updated"));
}
```

### Extract Entire Response

```java
@Test
public void testComplexFlow() {
    // Extract entire response object
    User createdUser = given()
        .contentType("application/json")
        .body(new User("Alice", "alice@example.com"))
        .when()
            .post("/users")
        .then()
            .statusCode(201)
            .extract()
            .as(User.class);  // Deserialize to POJO!

    // Now we have the full object
    assertNotNull(createdUser.getId());
    assertEquals("Alice", createdUser.getName());

    // Use it in next request
    given()
        .when()
            .delete("/users/" + createdUser.getId())
        .then()
            .statusCode(204);
}
```

### Extract Response for Detailed Assertions

```java
@Test
public void testDetailedValidation() {
    Response response = given()
        .when()
            .get("/users")
        .then()
            .statusCode(200)
            .extract()
            .response();

    // Multiple assertions on the same response
    assertEquals(200, response.statusCode());
    assertEquals("application/json", response.contentType());
    assertTrue(response.time() < 1000);  // Response time < 1 second

    List<User> users = response.jsonPath().getList("", User.class);
    assertEquals(3, users.size());
}
```

---

## Chapter 10: The Validation Problem — Advanced Assertions

### The Challenge

Complex validation scenarios:

```java
// API returns:
{
  "users": [
    {"id": 1, "name": "Alice", "age": 30},
    {"id": 2, "name": "Bob", "age": 25},
    {"id": 3, "name": "Charlie", "age": 35}
  ],
  "total": 3,
  "page": 1
}
```

How do you assert:
- All users have an age > 20?
- At least one user named "Bob"?
- Total matches array size?

### Solution 1: Hamcrest Matchers

```java
@Test
public void testComplexValidation() {
    given()
        .when()
            .get("/users")
        .then()
            .statusCode(200)
            .body("total", equalTo(3))
            .body("users.size()", equalTo(3))
            .body("users.name", hasItems("Alice", "Bob", "Charlie"))
            .body("users.age", everyItem(greaterThan(20)))
            .body("users.find { it.name == 'Bob' }.age", equalTo(25));
}
```

**Hamcrest matchers are powerful!**

Common matchers:
- `equalTo(value)` - Exact match
- `hasItems(...)` - Contains items
- `everyItem(matcher)` - All items match
- `greaterThan(value)` - Numeric comparison
- `containsString(text)` - String contains
- `notNullValue()` - Not null
- `hasSize(size)` - Collection size

### Solution 2: Multiple Assertions on Same Path

```java
@Test
public void testMultipleAssertions() {
    given()
        .when()
            .get("/users/1")
        .then()
            .statusCode(200)
            .body("name", equalTo("Alice"))
            .body("name", notNullValue())
            .body("name", containsString("Ali"))
            .body("email", endsWith("@example.com"))
            .body("age", allOf(greaterThan(18), lessThan(100)));
}
```

### Solution 3: Root Path for Cleaner Tests

```java
@Test
public void testWithRootPath() {
    given()
        .when()
            .get("/users/1")
        .then()
            .statusCode(200)
            .rootPath("user")  // Set root path
            .body("name", equalTo("Alice"))
            .body("email", equalTo("alice@example.com"))
            .body("address.city", equalTo("NYC"))
            .rootPath("user.address")  // Change root path
            .body("city", equalTo("NYC"))
            .body("zip", equalTo("10001"));
}
```

### Solution 4: Custom Assertions with Extract

```java
@Test
public void testCustomValidation() {
    ValidatableResponse response = given()
        .when()
            .get("/users")
        .then()
            .statusCode(200);

    // Extract and do custom validation
    List<Map<String, Object>> users = response.extract().path("users");

    // Complex business logic validation
    long activeUsers = users.stream()
        .filter(u -> "ACTIVE".equals(u.get("status")))
        .count();

    assertTrue(activeUsers >= users.size() * 0.8,
        "At least 80% of users should be active");

    // Continue with fluent assertions
    response
        .body("total", equalTo(users.size()))
        .body("users[0].name", notNullValue());
}
```

---

## Chapter 11: The Request Body Problem

### The Challenge

Sending different types of request bodies:
- JSON objects
- Form data
- Multipart files
- XML
- Custom content types

### Solution 1: JSON with POJO

```java
@Test
public void testCreateUserWithPojo() {
    User newUser = new User();
    newUser.setName("Alice");
    newUser.setEmail("alice@example.com");

    given()
        .contentType("application/json")
        .body(newUser)  // Automatically serialized to JSON!
        .when()
            .post("/users")
        .then()
            .statusCode(201);
}
```

### Solution 2: JSON with Map

```java
@Test
public void testCreateUserWithMap() {
    Map<String, Object> user = new HashMap<>();
    user.put("name", "Alice");
    user.put("email", "alice@example.com");
    user.put("age", 30);

    given()
        .contentType("application/json")
        .body(user)
        .when()
            .post("/users")
        .then()
            .statusCode(201);
}
```

### Solution 3: JSON String

```java
@Test
public void testCreateUserWithJsonString() {
    String json = """
        {
            "name": "Alice",
            "email": "alice@example.com",
            "address": {
                "city": "NYC",
                "zip": "10001"
            }
        }
        """;

    given()
        .contentType("application/json")
        .body(json)
        .when()
            .post("/users")
        .then()
            .statusCode(201);
}
```

### Solution 4: Form Parameters

```java
@Test
public void testLogin() {
    given()
        .contentType("application/x-www-form-urlencoded")
        .formParam("username", "alice@example.com")
        .formParam("password", "secret123")
        .when()
            .post("/auth/login")
        .then()
            .statusCode(200)
            .body("token", notNullValue());
}
```

### Solution 5: Multipart File Upload

```java
@Test
public void testFileUpload() {
    File file = new File("src/test/resources/test-file.txt");

    given()
        .multiPart("file", file)
        .multiPart("description", "Test file")
        .when()
            .post("/files/upload")
        .then()
            .statusCode(201)
            .body("filename", equalTo("test-file.txt"));
}
```

### Solution 6: Complex Multipart

```java
@Test
public void testComplexMultipart() {
    File avatar = new File("avatar.png");
    User userData = new User("Alice", "alice@example.com");

    given()
        .multiPart("avatar", avatar, "image/png")
        .multiPart("user", userData, "application/json")
        .when()
            .post("/users/register")
        .then()
            .statusCode(201);
}
```

---

## Chapter 12: The Logging and Debugging Problem

### The Challenge

When tests fail, you need to see:
- What request was sent?
- What response was received?
- What went wrong?

### Solution 1: Basic Logging

```java
@Test
public void testWithLogging() {
    given()
        .log().all()  // Log entire request
        .when()
            .get("/users/1")
        .then()
            .log().all()  // Log entire response
            .statusCode(200);
}

// Output:
// Request method:	GET
// Request URI:	http://localhost:8080/users/1
// Headers:		Accept=*/*
// Cookies:		<none>
// Body:		<none>
//
// HTTP/1.1 200 OK
// Content-Type: application/json
// Body:
// {"id":1,"name":"Alice","email":"alice@example.com"}
```

### Solution 2: Conditional Logging

```java
@Test
public void testConditionalLogging() {
    given()
        .log().ifValidationFails()  // Only log if test fails
        .when()
            .get("/users/1")
        .then()
            .log().ifError()  // Only log if status >= 400
            .statusCode(200);
}
```

### Solution 3: Selective Logging

```java
@Test
public void testSelectiveLogging() {
    given()
        .log().headers()  // Only log request headers
        .when()
            .get("/users/1")
        .then()
            .log().body()  // Only log response body
            .statusCode(200);
}
```

### Solution 4: Enable Logging for All Tests

```java
@SpringBootTest
public abstract class ApiTestBase {

    @BeforeEach
    public void setUp() {
        RestAssured.filters(new RequestLoggingFilter(), new ResponseLoggingFilter());
    }
}

// Now all tests log automatically!
```

### Solution 5: Pretty Print JSON

```java
@Test
public void testPrettyPrint() {
    String prettyJson = given()
        .when()
            .get("/users")
        .then()
            .extract()
            .response()
            .prettyPrint();  // Pretty-printed JSON

    System.out.println(prettyJson);
    // Output:
    // {
    //   "users": [
    //     {
    //       "id": 1,
    //       "name": "Alice"
    //     }
    //   ]
    // }
}
```

---

## Chapter 13: The Performance Testing Problem

### The Challenge

You need to verify:
- Response time is acceptable
- API handles load
- No performance regressions

### Solution 1: Response Time Assertion

```java
@Test
public void testResponseTime() {
    given()
        .when()
            .get("/users")
        .then()
            .time(lessThan(500L), TimeUnit.MILLISECONDS)  // Must respond < 500ms
            .statusCode(200);
}
```

### Solution 2: Measure and Extract Time

```java
@Test
public void testAndMeasureTime() {
    long responseTime = given()
        .when()
            .get("/users")
        .then()
            .statusCode(200)
            .extract()
            .time();  // Extract response time

    System.out.println("Response time: " + responseTime + "ms");
    assertTrue(responseTime < 1000, "Response should be under 1 second");
}
```

### Solution 3: Performance Test with Loop

```java
@Test
public void testPerformanceUnderLoad() {
    List<Long> responseTimes = new ArrayList<>();

    for (int i = 0; i < 100; i++) {
        long time = given()
            .when()
                .get("/users")
            .then()
                .statusCode(200)
                .extract()
                .time();

        responseTimes.add(time);
    }

    // Calculate statistics
    double avgTime = responseTimes.stream()
        .mapToLong(Long::longValue)
        .average()
        .orElse(0);

    long maxTime = responseTimes.stream()
        .mapToLong(Long::longValue)
        .max()
        .orElse(0);

    System.out.println("Average response time: " + avgTime + "ms");
    System.out.println("Max response time: " + maxTime + "ms");

    assertTrue(avgTime < 200, "Average response time should be under 200ms");
    assertTrue(maxTime < 500, "Max response time should be under 500ms");
}
```

---

## Chapter 14: The Schema Validation Problem

### The Challenge

Ensure API responses match expected structure:
- Required fields present
- Correct data types
- Enum values valid
- Nested structure correct

### Solution: JSON Schema Validation

```java
@Test
public void testJsonSchema() {
    given()
        .when()
            .get("/users/1")
        .then()
            .statusCode(200)
            .body(matchesJsonSchemaInClasspath("user-schema.json"));
}
```

**user-schema.json**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["id", "name", "email"],
  "properties": {
    "id": {
      "type": "integer"
    },
    "name": {
      "type": "string",
      "minLength": 1
    },
    "email": {
      "type": "string",
      "format": "email"
    },
    "age": {
      "type": "integer",
      "minimum": 0,
      "maximum": 150
    },
    "status": {
      "type": "string",
      "enum": ["ACTIVE", "INACTIVE", "PENDING"]
    }
  }
}
```

**Benefits**:
- Validates structure automatically
- Ensures required fields present
- Type safety
- Enum validation
- Easy to maintain

### Inline Schema Validation

```java
@Test
public void testInlineSchema() {
    given()
        .when()
            .get("/users")
        .then()
            .statusCode(200)
            .body(matchesJsonSchema("""
                {
                  "type": "object",
                  "required": ["users", "total"],
                  "properties": {
                    "users": {
                      "type": "array",
                      "items": {
                        "type": "object",
                        "required": ["id", "name"],
                        "properties": {
                          "id": {"type": "integer"},
                          "name": {"type": "string"}
                        }
                      }
                    },
                    "total": {"type": "integer"}
                  }
                }
                """));
}
```

---

## Chapter 15: The Real-World Testing Patterns

### Pattern 1: Page-Based User Tests

```java
public class UserApiTest extends ApiTestBase {

    @Test
    public void testGetUserList() {
        given()
            .queryParam("page", 1)
            .queryParam("size", 10)
            .when()
                .get("/users")
            .then()
                .statusCode(200)
                .body("users.size()", lessThanOrEqualTo(10))
                .body("page", equalTo(1))
                .body("totalPages", greaterThanOrEqualTo(1));
    }

    @Test
    public void testSearchUsers() {
        given()
            .queryParam("name", "Alice")
            .queryParam("status", "ACTIVE")
            .when()
                .get("/users/search")
            .then()
                .statusCode(200)
                .body("users.name", everyItem(containsString("Alice")))
                .body("users.status", everyItem(equalTo("ACTIVE")));
    }
}
```

### Pattern 2: CRUD Test Flow

```java
public class UserCrudTest extends ApiTestBase {

    @Test
    public void testCompleteUserLifecycle() {
        // CREATE
        Long userId = given()
            .contentType("application/json")
            .body(Map.of(
                "name", "Alice",
                "email", "alice@example.com"
            ))
            .when()
                .post("/users")
            .then()
                .statusCode(201)
                .body("name", equalTo("Alice"))
                .extract()
                .path("id");

        // READ
        given()
            .when()
                .get("/users/" + userId)
            .then()
                .statusCode(200)
                .body("name", equalTo("Alice"))
                .body("email", equalTo("alice@example.com"));

        // UPDATE
        given()
            .contentType("application/json")
            .body(Map.of("name", "Alice Updated"))
            .when()
                .put("/users/" + userId)
            .then()
                .statusCode(200)
                .body("name", equalTo("Alice Updated"));

        // Verify update
        given()
            .when()
                .get("/users/" + userId)
            .then()
                .statusCode(200)
                .body("name", equalTo("Alice Updated"));

        // DELETE
        given()
            .when()
                .delete("/users/" + userId)
            .then()
                .statusCode(204);

        // Verify deletion
        given()
            .when()
                .get("/users/" + userId)
            .then()
                .statusCode(404);
    }
}
```

### Pattern 3: Error Handling Tests

```java
public class UserErrorTest extends ApiTestBase {

    @Test
    public void testUserNotFound() {
        given()
            .when()
                .get("/users/99999")
            .then()
                .statusCode(404)
                .body("error", equalTo("User not found"))
                .body("message", containsString("No user with ID 99999"));
    }

    @Test
    public void testInvalidInput() {
        given()
            .contentType("application/json")
            .body(Map.of(
                "name", "",  // Invalid: empty name
                "email", "invalid-email"  // Invalid: bad format
            ))
            .when()
                .post("/users")
            .then()
                .statusCode(400)
                .body("errors.name", hasItem("Name cannot be empty"))
                .body("errors.email", hasItem("Email must be valid"));
    }

    @Test
    public void testUnauthorized() {
        given()
            // No auth header
            .when()
                .get("/users/me")
            .then()
                .statusCode(401)
                .body("error", equalTo("Unauthorized"));
    }

    @Test
    public void testForbidden() {
        String regularUserToken = getTokenForUser("user@example.com");

        given()
            .header("Authorization", "Bearer " + regularUserToken)
            .when()
                .delete("/admin/users/1")  // Admin-only endpoint
            .then()
                .statusCode(403)
                .body("error", equalTo("Forbidden"));
    }
}
```

### Pattern 4: Integration with Database

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Transactional  // Rollback after each test
public class UserIntegrationTest extends ApiTestBase {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private EmailService emailService;  // Mock this!

    @MockBean
    private EmailService emailServiceMock;

    @Test
    public void testUserCreationSendsEmail() {
        // Arrange
        when(emailServiceMock.sendWelcomeEmail(any()))
            .thenReturn(true);

        // Act
        Long userId = given()
            .contentType("application/json")
            .body(Map.of(
                "name", "Alice",
                "email", "alice@example.com"
            ))
            .when()
                .post("/users")
            .then()
                .statusCode(201)
                .extract()
                .path("id");

        // Assert
        verify(emailServiceMock).sendWelcomeEmail("alice@example.com");

        // Verify in database
        Optional<User> user = userRepository.findById(userId);
        assertTrue(user.isPresent());
        assertEquals("Alice", user.get().getName());
    }
}
```

---

## Chapter 16: The Philosophy of Rest Assured

### Principle 1: Readability Over Brevity

**Bad** (RestTemplate):
```java
ResponseEntity<Map> r = rt.exchange(url, HttpMethod.GET, new HttpEntity<>(h), Map.class);
assertEquals(200, r.getStatusCodeValue());
assertEquals("Alice", ((Map)r.getBody().get("user")).get("name"));
```

**Good** (Rest Assured):
```java
given()
    .when()
        .get("/users/1")
    .then()
        .statusCode(200)
        .body("user.name", equalTo("Alice"));
```

**Tests are documentation!** They should read like specifications.

### Principle 2: Fluent APIs Express Intent

**Compare**:

```java
// Imperative (what + how)
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestMethod("POST");
conn.setRequestProperty("Content-Type", "application/json");
conn.setDoOutput(true);
// ... more boilerplate

// Declarative (what, not how)
given()
    .contentType("application/json")
    .body(user)
    .post("/users");
```

**Fluent APIs hide "how" and emphasize "what".**

### Principle 3: BDD Style Improves Communication

```java
// Technical test
@Test
public void test1() {
    Response r = httpClient.get("/users?status=active");
    assertEquals(200, r.code);
    assertTrue(r.body.contains("users"));
}

// BDD test (self-documenting)
@Test
public void shouldReturnOnlyActiveUsers() {
    given()
        .queryParam("status", "active")
        .when()
            .get("/users")
        .then()
            .statusCode(200)
            .body("users.status", everyItem(equalTo("ACTIVE")));
}
```

**Product managers can read the second test!**

### Principle 4: Separation of Concerns

**Rest Assured separates**:
1. **Request building** (`given()`) - Setup
2. **Action** (`when()`) - Execution
3. **Verification** (`then()`) - Assertions

**This mirrors human thinking**:
- "Given these conditions..."
- "When I do this action..."
- "Then I expect this result..."

### Principle 5: Flexibility Through Composition

```java
// Reusable request specs
RequestSpecification authSpec = new RequestSpecBuilder()
    .addHeader("Authorization", "Bearer " + token)
    .build();

RequestSpecification jsonSpec = new RequestSpecBuilder()
    .setContentType(ContentType.JSON)
    .build();

// Compose them
given()
    .spec(authSpec)
    .spec(jsonSpec)
    .body(data)
    .post("/users");
```

**Build complex behavior from simple pieces.**

---

## Chapter 17: Common Mistakes and How to Avoid Them

### Mistake 1: Not Using Base Test Class

**Wrong**:
```java
public class UserApiTest {
    @Test
    public void test1() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = 8080;
        // Test code...
    }

    @Test
    public void test2() {
        RestAssured.baseURI = "http://localhost";  // Repeated!
        RestAssured.port = 8080;
        // Test code...
    }
}
```

**Right**:
```java
public abstract class ApiTestBase {
    @LocalServerPort
    private int port;

    @BeforeEach
    public void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api";
    }
}

public class UserApiTest extends ApiTestBase {
    @Test
    public void test1() {
        // Clean test code
    }
}
```

### Mistake 2: Hard-Coding Test Data

**Wrong**:
```java
@Test
public void test() {
    given()
        .when()
            .get("/users/1")  // What if ID 1 doesn't exist?
        .then()
            .body("name", equalTo("Alice"));  // Brittle assumption
}
```

**Right**:
```java
@Test
public void test() {
    User user = createUser("Alice", "alice@example.com");

    given()
        .when()
            .get("/users/" + user.getId())
        .then()
            .body("name", equalTo("Alice"));
}
```

### Mistake 3: Testing Multiple Things in One Test

**Wrong**:
```java
@Test
public void testEverything() {
    // Create user
    Long id = given().body(user).post("/users").extract().path("id");

    // Get user
    given().get("/users/" + id).then().statusCode(200);

    // Update user
    given().body(update).put("/users/" + id).then().statusCode(200);

    // Delete user
    given().delete("/users/" + id).then().statusCode(204);

    // If any step fails, hard to debug!
}
```

**Right**:
```java
@Test
public void testCreateUser() {
    given()
        .body(user)
        .when()
            .post("/users")
        .then()
            .statusCode(201);
}

@Test
public void testGetUser() {
    User user = createUser();

    given()
        .when()
            .get("/users/" + user.getId())
        .then()
            .statusCode(200);
}

// Separate tests for each operation
```

### Mistake 4: Not Cleaning Up Test Data

**Wrong**:
```java
@Test
public void test1() {
    createUser("Alice", "alice@example.com");
    // Data left in database!
}

@Test
public void test2() {
    // Might conflict with test1's data
}
```

**Right**:
```java
@BeforeEach
public void cleanDatabase() {
    userRepository.deleteAll();
}

@Test
public void test1() {
    createUser("Alice", "alice@example.com");
    // Will be cleaned before next test
}
```

### Mistake 5: Not Logging on Failure

**Wrong**:
```java
@Test
public void test() {
    given()
        .when()
            .get("/users/1")
        .then()
            .statusCode(200);
    // When it fails, no idea what the response was!
}
```

**Right**:
```java
@Test
public void test() {
    given()
        .log().ifValidationFails()
        .when()
            .get("/users/1")
        .then()
            .log().ifError()
            .statusCode(200);
}
```

---

## Chapter 18: Putting It All Together

### The Complete Example

```java
// Base class for all API tests
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Transactional
public abstract class ApiTestBase {

    @LocalServerPort
    protected int port;

    @Autowired
    protected UserRepository userRepository;

    protected RequestSpecification authenticatedSpec;

    @BeforeEach
    public void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api";

        // Clean database
        userRepository.deleteAll();

        // Setup authenticated request spec
        String token = getAuthToken();
        authenticatedSpec = new RequestSpecBuilder()
            .addHeader("Authorization", "Bearer " + token)
            .setContentType(ContentType.JSON)
            .build();

        // Enable logging on failure
        RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();
    }

    @AfterEach
    public void tearDown() {
        RestAssured.reset();
    }

    protected String getAuthToken() {
        return given()
            .contentType("application/x-www-form-urlencoded")
            .formParam("username", "test@example.com")
            .formParam("password", "password")
            .when()
                .post("/auth/login")
            .then()
                .statusCode(200)
                .extract()
                .path("token");
    }

    protected User createUser(String name, String email) {
        User user = new User();
        user.setName(name);
        user.setEmail(email);
        return userRepository.save(user);
    }
}

// Actual test class
public class UserApiTest extends ApiTestBase {

    @Test
    public void shouldGetUserById() {
        // Arrange
        User alice = createUser("Alice", "alice@example.com");

        // Act & Assert
        given()
            .spec(authenticatedSpec)
            .when()
                .get("/users/" + alice.getId())
            .then()
                .statusCode(200)
                .body("id", equalTo(alice.getId().intValue()))
                .body("name", equalTo("Alice"))
                .body("email", equalTo("alice@example.com"));
    }

    @Test
    public void shouldCreateUser() {
        // Arrange
        Map<String, Object> newUser = Map.of(
            "name", "Bob",
            "email", "bob@example.com"
        );

        // Act & Assert
        Long userId = given()
            .spec(authenticatedSpec)
            .body(newUser)
            .when()
                .post("/users")
            .then()
                .statusCode(201)
                .body("name", equalTo("Bob"))
                .body("email", equalTo("bob@example.com"))
                .extract()
                .path("id");

        // Verify in database
        Optional<User> user = userRepository.findById(userId);
        assertTrue(user.isPresent());
        assertEquals("Bob", user.get().getName());
    }

    @Test
    public void shouldReturnPagedUsers() {
        // Arrange
        createUser("Alice", "alice@example.com");
        createUser("Bob", "bob@example.com");
        createUser("Charlie", "charlie@example.com");

        // Act & Assert
        given()
            .spec(authenticatedSpec)
            .queryParam("page", 0)
            .queryParam("size", 2)
            .when()
                .get("/users")
            .then()
                .statusCode(200)
                .body("content.size()", equalTo(2))
                .body("totalElements", equalTo(3))
                .body("totalPages", equalTo(2))
                .body("content.name", hasItems("Alice", "Bob"));
    }

    @Test
    public void shouldReturn404ForNonexistentUser() {
        given()
            .spec(authenticatedSpec)
            .when()
                .get("/users/99999")
            .then()
                .statusCode(404)
                .body("error", equalTo("Not Found"))
                .body("message", containsString("User not found"));
    }

    @Test
    public void shouldValidateEmailFormat() {
        Map<String, Object> invalidUser = Map.of(
            "name", "Invalid",
            "email", "not-an-email"
        );

        given()
            .spec(authenticatedSpec)
            .body(invalidUser)
            .when()
                .post("/users")
            .then()
                .statusCode(400)
                .body("errors.email", hasItem(containsString("valid email")));
    }

    @Test
    public void shouldUpdateUser() {
        // Arrange
        User alice = createUser("Alice", "alice@example.com");

        Map<String, Object> update = Map.of("name", "Alice Updated");

        // Act & Assert
        given()
            .spec(authenticatedSpec)
            .body(update)
            .when()
                .put("/users/" + alice.getId())
            .then()
                .statusCode(200)
                .body("name", equalTo("Alice Updated"))
                .body("email", equalTo("alice@example.com"));  // Unchanged

        // Verify in database
        User updated = userRepository.findById(alice.getId()).get();
        assertEquals("Alice Updated", updated.getName());
    }

    @Test
    public void shouldDeleteUser() {
        // Arrange
        User alice = createUser("Alice", "alice@example.com");

        // Act
        given()
            .spec(authenticatedSpec)
            .when()
                .delete("/users/" + alice.getId())
            .then()
                .statusCode(204);

        // Assert - verify deleted
        given()
            .spec(authenticatedSpec)
            .when()
                .get("/users/" + alice.getId())
            .then()
                .statusCode(404);

        // Verify in database
        assertFalse(userRepository.findById(alice.getId()).isPresent());
    }
}
```

---

## Conclusion: You Just Invented Rest Assured

### The Journey

You started with a simple problem: **How do we test REST APIs in Java?**

You tried:
1. **Manual cURL** (not automated)
2. **HttpURLConnection** (too verbose)
3. **RestTemplate** (better, but still clunky)

You invented:
1. **Fluent API** (method chaining for readability)
2. **BDD style** (given/when/then)
3. **JSON Path** (flexible response validation)
4. **Request specifications** (reusable configuration)
5. **Response extraction** (chain requests)
6. **Hamcrest matchers** (powerful assertions)
7. **Logging** (debugging support)

**You didn't just learn Rest Assured. You understand *why* it exists.**

### The Meta-Lesson

**Every framework solves pain points**:

1. **Identify the problem** - Testing REST APIs is tedious
2. **Try the naive solution** - HttpURLConnection is too verbose
3. **Iterate with improvements** - RestTemplate, then Rest Assured
4. **Appreciate the trade-offs** - Readability vs. power vs. simplicity

### The Big Picture

```
Manual Testing (Postman)
    ↓
HttpURLConnection (too low-level)
    ↓
RestTemplate (better, but still verbose)
    ↓
Rest Assured (fluent, readable, powerful)
    ↓
Your API tests are now:
  ✓ Automated
  ✓ Readable
  ✓ Maintainable
  ✓ In your codebase
  ✓ Part of CI/CD
```

### Next Steps

**Practice**:
- Test your own Spring Boot APIs
- Write integration tests for CRUD operations
- Add authentication to your tests
- Practice JSON path queries
- Use schema validation

**Explore**:
- Advanced Hamcrest matchers
- Custom filters
- OAuth2 authentication
- File upload/download testing
- WebSocket testing (Rest Assured has support!)

**Think**:
- How would you test GraphQL APIs?
- How would you test gRPC?
- What if responses were XML instead of JSON?
- How would you handle rate limiting in tests?

### Final Thought

**Rest Assured is not just a testing framework. It's a case study in:**
- API design (fluent interfaces)
- Developer experience (readability)
- Separation of concerns (given/when/then)
- Composition (request specs)
- Extensibility (filters, matchers)

**You now understand the "why" behind every Rest Assured feature.**

**Go forth and test those APIs. You've earned it.**

---

## Appendix A: Quick Reference

### Basic GET Test

```java
given()
    .when()
        .get("/users/1")
    .then()
        .statusCode(200)
        .body("name", equalTo("Alice"));
```

### POST with JSON Body

```java
given()
    .contentType("application/json")
    .body(Map.of("name", "Alice", "email", "alice@example.com"))
    .when()
        .post("/users")
    .then()
        .statusCode(201);
```

### With Authentication

```java
given()
    .header("Authorization", "Bearer " + token)
    .when()
        .get("/users/me")
    .then()
        .statusCode(200);
```

### With Query Parameters

```java
given()
    .queryParam("page", 0)
    .queryParam("size", 10)
    .queryParam("sort", "name,asc")
    .when()
        .get("/users")
    .then()
        .statusCode(200);
```

### Extract Response Data

```java
Long userId = given()
    .body(user)
    .when()
        .post("/users")
    .then()
        .statusCode(201)
        .extract()
        .path("id");
```

### JSON Path Assertions

```java
.body("user.name", equalTo("Alice"))
.body("user.emails[0]", equalTo("alice@example.com"))
.body("users.size()", equalTo(3))
.body("users.name", hasItems("Alice", "Bob"))
.body("users.age", everyItem(greaterThan(18)))
```

### Request Specification

```java
RequestSpecification authSpec = new RequestSpecBuilder()
    .addHeader("Authorization", "Bearer " + token)
    .setContentType(ContentType.JSON)
    .setBaseUri("http://localhost")
    .setPort(8080)
    .build();

given()
    .spec(authSpec)
    .when()
        .get("/users")
    .then()
        .statusCode(200);
```

### Logging

```java
given()
    .log().all()  // Log request
    .when()
        .get("/users")
    .then()
        .log().all()  // Log response
        .statusCode(200);
```

---

## Appendix B: Common Hamcrest Matchers

### Equality
```java
equalTo(value)              // Exact match
not(equalTo(value))         // Not equal
```

### Strings
```java
containsString("text")      // Contains substring
startsWith("prefix")        // Starts with
endsWith("suffix")          // Ends with
```

### Numbers
```java
greaterThan(10)             // > 10
greaterThanOrEqualTo(10)    // >= 10
lessThan(10)                // < 10
lessThanOrEqualTo(10)       // <= 10
```

### Collections
```java
hasItems("a", "b")          // Contains items
hasItem("a")                // Contains single item
hasSize(3)                  // Size equals
everyItem(matcher)          // All items match
empty()                     // Is empty
```

### Null checks
```java
notNullValue()              // Not null
nullValue()                 // Is null
```

### Combining
```java
allOf(matcher1, matcher2)   // Both match
anyOf(matcher1, matcher2)   // Either matches
```

---

## Appendix C: Spring Boot Test Setup

### Maven Dependencies

```xml
<dependencies>
    <!-- Spring Boot Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Rest Assured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.3.0</version>
        <scope>test</scope>
    </dependency>

    <!-- JSON Schema Validator -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>json-schema-validator</artifactId>
        <version>5.3.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle Dependencies

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'io.rest-assured:rest-assured:5.3.0'
    testImplementation 'io.rest-assured:json-schema-validator:5.3.0'
}
```

---

**Version**: 1.0
**Date**: November 2025
**License**: Creative Commons BY-SA 4.0

---

*"The best way to understand a framework is to imagine inventing it yourself."*

**Thank you for taking this journey. Now you don't just use Rest Assured—you understand it.**
