Awesome—here’s a **production-ready, generic Cucumber test setup** for Spring REST APIs that covers **GET/POST/PUT/PATCH/DELETE** with reusable steps, data-driven assertions, headers (incl. JWT), query/path params, and JSON schema checks. It uses **JUnit 5 + Cucumber + Rest Assured** (works great both against a running service or a Testcontainers-booted one). You can drop this into any Maven Spring project.

---

# 1) Maven setup

```xml
<!-- pom.xml -->
<project>
  <properties>
    <java.version>17</java.version>
    <cucumber.version>7.18.1</cucumber.version>
    <restassured.version>5.5.0</restassured.version>
    <junit.jupiter.version>5.10.2</junit.jupiter.version>
  </properties>

  <dependencies>
    <!-- Cucumber + JUnit 5 -->
    <dependency>
      <groupId>io.cucumber</groupId>
      <artifactId>cucumber-java</artifactId>
      <version>${cucumber.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.cucumber</groupId>
      <artifactId>cucumber-junit-platform-engine</artifactId>
      <version>${cucumber.version}</version>
      <scope>test</scope>
    </dependency>

    <!-- Rest Assured for HTTP + JSONPath -->
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <version>${restassured.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>json-schema-validator</artifactId>
      <version>${restassured.version}</version>
      <scope>test</scope>
    </dependency>

    <!-- Assertions -->
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <version>3.26.0</version>
      <scope>test</scope>
    </dependency>

    <!-- Jackson for object mapping (optional) -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.17.2</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <!-- Surefire: runs Cucumber via JUnit Platform -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.2.5</version>
        <configuration>
          <includes>
            <include>**/*CucumberIT.java</include>
          </includes>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

---

# 2) Project layout

```
src
└── test
    ├── java
    │   └── com.example.api
    │       ├── config
    │       │   └── TestConfig.java
    │       ├── support
    │       │   ├── TestContext.java
    │       │   ├── ApiClient.java
    │       │   └── Hooks.java
    │       ├── steps
    │       │   └── GenericApiSteps.java
    │       └── CucumberIT.java
    └── resources
        ├── application-test.properties
        ├── features
        │   └── generic_api.feature
        └── schemas
            └── user-list.schema.json
```

---

# 3) Config & Context

```java
// src/test/java/com/example/api/config/TestConfig.java
package com.example.api.config;

public final class TestConfig {
  private TestConfig() {}
  public static String baseUrl() {
    // priority: system property > env var > properties file
    String fromSys = System.getProperty("api.baseUrl");
    if (fromSys != null && !fromSys.isBlank()) return fromSys;
    String fromEnv = System.getenv("API_BASE_URL");
    if (fromEnv != null && !fromEnv.isBlank()) return fromEnv;
    // fallback for local dev
    return "http://localhost:8080";
  }
  public static String defaultJwt() {
    String fromSys = System.getProperty("api.jwt");
    if (fromSys != null) return fromSys;
    String fromEnv = System.getenv("API_JWT");
    return fromEnv; // may be null; steps can set later
  }
}
```

```java
// src/test/java/com/example/api/support/TestContext.java
package com.example.api.support;

import io.restassured.response.Response;
import java.util.HashMap;
import java.util.Map;

public class TestContext {
  private String jwt;                         // bearer token
  private final Map<String, Object> vars = new HashMap<>(); // store ids, etc.
  private Response lastResponse;

  public String getJwt() { return jwt; }
  public void setJwt(String jwt) { this.jwt = jwt; }

  public void putVar(String key, Object value) { vars.put(key, value); }
  public Object getVar(String key) { return vars.get(key); }

  public Response getLastResponse() { return lastResponse; }
  public void setLastResponse(Response resp) { this.lastResponse = resp; }

  public void clear() {
    vars.clear();
    lastResponse = null;
    jwt = null;
  }
}
```

```java
// src/test/java/com/example/api/support/ApiClient.java
package com.example.api.support;

import io.restassured.builder.RequestSpecBuilder;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;

import static io.restassured.RestAssured.given;

import java.util.Map;

public class ApiClient {

  private final String baseUrl;
  private final TestContext ctx;

  public ApiClient(String baseUrl, TestContext ctx) {
    this.baseUrl = baseUrl;
    this.ctx = ctx;
  }

  private RequestSpecification spec(Map<String, String> headers, Map<String, ?> qs) {
    RequestSpecBuilder b = new RequestSpecBuilder()
        .setBaseUri(baseUrl)
        .setContentType(ContentType.JSON)
        .setAccept(ContentType.JSON);

    if (ctx.getJwt() != null && !ctx.getJwt().isBlank()) {
      b.addHeader("Authorization", "Bearer " + ctx.getJwt());
    }
    if (headers != null) headers.forEach(b::addHeader);
    if (qs != null) b.addQueryParams(qs);

    return b.build();
  }

  public io.restassured.response.Response request(
      String method, String path,
      Map<String, String> headers,
      Map<String, ?> queryParams,
      String body) {

    RequestSpecification s = given().spec(spec(headers, queryParams));
    if (body != null && !body.isBlank()) s = s.body(body);

    return switch (method.toUpperCase()) {
      case "GET"    -> s.when().get(path);
      case "POST"   -> s.when().post(path);
      case "PUT"    -> s.when().put(path);
      case "PATCH"  -> s.when().patch(path);
      case "DELETE" -> s.when().delete(path);
      default -> throw new IllegalArgumentException("Unsupported method: " + method);
    };
  }
}
```

```java
// src/test/java/com/example/api/support/Hooks.java
package com.example.api.support;

import com.example.api.config.TestConfig;
import io.cucumber.java.After;
import io.cucumber.java.Before;

public class Hooks {
  public static final ThreadLocal<TestContext> CTX = ThreadLocal.withInitial(TestContext::new);
  public static final ThreadLocal<ApiClient> CLIENT = new ThreadLocal<>();

  @Before
  public void beforeScenario() {
    TestContext ctx = CTX.get();
    ctx.clear();
    ctx.setJwt(TestConfig.defaultJwt()); // optional
    CLIENT.set(new ApiClient(TestConfig.baseUrl(), ctx));
  }

  @After
  public void afterScenario() {
    CTX.remove();
    CLIENT.remove();
  }
}
```

---

# 4) Generic step definitions

```java
// src/test/java/com/example/api/steps/GenericApiSteps.java
package com.example.api.steps;

import com.example.api.support.Hooks;
import com.example.api.support.TestContext;
import com.example.api.support.ApiClient;

import io.cucumber.java.en.*;
import io.cucumber.datatable.DataTable;

import io.restassured.module.jsv.JsonSchemaValidator;
import io.restassured.response.Response;

import org.assertj.core.api.Assertions;

import java.io.File;
import java.util.*;
import java.util.stream.Collectors;

public class GenericApiSteps {

  private TestContext ctx() { return Hooks.CTX.get(); }
  private ApiClient client() { return Hooks.CLIENT.get(); }

  @Given("the API base URL is {string}")
  public void apiBaseUrl_is(String baseUrl) {
    // override base URL at runtime if needed
    Hooks.CLIENT.set(new ApiClient(baseUrl, ctx()));
  }

  @Given("I set the JWT token {string}")
  public void iSetJwt(String jwt) {
    ctx().setJwt(resolve(jwt));
  }

  @Given("I store {string} as {string}")
  public void iStoreVar(String value, String key) {
    ctx().putVar(key, resolve(value));
  }

  @When("I send a {word} request to {string}")
  public void iSendRequestNoBody(String method, String path) {
    send(method, path, Collections.emptyMap(), Collections.emptyMap(), null);
  }

  @When("I send a {word} request to {string} with body:")
  public void iSendRequestWithBody(String method, String path, String body) {
    send(method, path, Collections.emptyMap(), Collections.emptyMap(), resolve(body));
  }

  @When("I send a {word} request to {string} with headers:")
  public void iSendRequestWithHeaders(String method, String path, DataTable table) {
    Map<String, String> headers = table.asMap(String.class, String.class)
        .entrySet().stream()
        .collect(Collectors.toMap(Map.Entry::getKey, e -> resolve(e.getValue())));
    send(method, path, headers, Collections.emptyMap(), null);
  }

  @When("I send a {word} request to {string} with query params:")
  public void iSendRequestWithQuery(String method, String path, DataTable table) {
    Map<String, String> q = table.asMap(String.class, String.class)
        .entrySet().stream()
        .collect(Collectors.toMap(Map.Entry::getKey, e -> resolve(e.getValue())));
    send(method, path, Collections.emptyMap(), q, null);
  }

  @When("I send a {word} request to {string} with headers and body:")
  public void iSendRequestWithHeadersAndBody(String method, String path, DataTable headers, String body) {
    Map<String, String> h = headers.asMap(String.class, String.class)
        .entrySet().stream()
        .collect(Collectors.toMap(Map.Entry::getKey, e -> resolve(e.getValue())));
    send(method, path, h, Collections.emptyMap(), resolve(body));
  }

  @Then("the response status should be {int}")
  public void responseStatusShouldBe(int code) {
    Assertions.assertThat(ctx().getLastResponse().statusCode())
        .as("HTTP status").isEqualTo(code);
  }

  @Then("the response header {string} should be {string}")
  public void responseHeaderShouldBe(String header, String value) {
    String actual = ctx().getLastResponse().getHeader(header);
    Assertions.assertThat(actual).as("Header " + header).isEqualTo(resolve(value));
  }

  @Then("the JSON at path {string} should be {string}")
  public void jsonAtPathShouldBe(String jsonPath, String expected) {
    Object actual = ctx().getLastResponse().jsonPath().get(jsonPath);
    Assertions.assertThat(String.valueOf(actual)).isEqualTo(resolve(expected));
  }

  @Then("the JSON at path {string} should exist")
  public void jsonPathExists(String jsonPath) {
    Object val = ctx().getLastResponse().jsonPath().get(jsonPath);
    Assertions.assertThat(val).as("JSON path " + jsonPath).isNotNull();
  }

  @Then("the JSON at path {string} should match regex {string}")
  public void jsonPathMatchesRegex(String jsonPath, String regex) {
    Object val = ctx().getLastResponse().jsonPath().get(jsonPath);
    Assertions.assertThat(String.valueOf(val)).matches(resolve(regex));
  }

  @Then("the response should match JSON schema {string}")
  public void jsonSchema(String schemaName) {
    File schema = new File("src/test/resources/schemas/" + schemaName);
    ctx().getLastResponse().then().assertThat().body(JsonSchemaValidator.matchesJsonSchema(schema));
  }

  @Then("I save JSON path {string} as {string}")
  public void saveJsonPathAs(String jsonPath, String key) {
    Object val = ctx().getLastResponse().jsonPath().get(jsonPath);
    ctx().putVar(key, val);
  }

  // Util

  private void send(String method, String path, Map<String, String> headers,
                    Map<String, ?> query, String body) {

    String resolvedPath = resolve(path);
    Response resp = client().request(method, resolvedPath, headers, query, body);
    ctx().setLastResponse(resp);
  }

  private String resolve(String text) {
    if (text == null) return null;
    // allows placeholders like ${userId} or ${token}
    String out = text;
    for (Map.Entry<String, Object> e : ctx().getClass()
        .cast(ctx()).getClass().getDeclaredFields() /* dummy to avoid warning */; /* no-op */) {
      // not used; keep simple variable replacement via map only
    }
    for (Map.Entry<String, Object> e : snapshotVars().entrySet()) {
      out = out.replace("${" + e.getKey() + "}", String.valueOf(e.getValue()));
    }
    return out;
  }

  private Map<String, Object> snapshotVars() {
    // expose TestContext vars via reflection-safe copy
    try {
      var f = TestContext.class.getDeclaredField("vars");
      f.setAccessible(true);
      Map<String, Object> map = (Map<String, Object>) f.get(Hooks.CTX.get());
      return new HashMap<>(map);
    } catch (Exception e) {
      return Map.of();
    }
  }
}
```

> The `resolve(...)` lets you refer to stored values (IDs, tokens) in later steps using `${varName}`.

---

# 5) Cucumber runner

```java
// src/test/java/com/example/api/CucumberIT.java
package com.example.api;

import io.cucumber.junit.platform.engine.Cucumber;

@Cucumber
public class CucumberIT {
  // Empty: Cucumber + JUnit Platform will pick up feature files under src/test/resources/features
}
```

---

# 6) A single generic, data-driven feature

```gherkin
# src/test/resources/features/generic_api.feature
Feature: Generic API Contract & Behavior

  # Optional: override base URL here (else use API_BASE_URL or http://localhost:8080)
  Background:
    Given the API base URL is "http://localhost:8080"
    And I set the JWT token "${jwtToken}"   # can be injected via system/env or set later

  @get_list
  Scenario: GET list of users with paging
    When I send a GET request to "/api/users?size=5&page=0"
    Then the response status should be 200
    And the response header "Content-Type" should be "application/json"
    And the JSON at path "content.size()" should be "5"
    And the JSON at path "pageable.pageNumber" should be "0"
    And the response should match JSON schema "user-list.schema.json"

  @create_update_delete
  Scenario: Create, update, and delete a user (POST -> PUT -> DELETE)
    When I send a POST request to "/api/users" with body:
      """
      {
        "name": "Ravi Test",
        "email": "ravi.${rand}@example.com",
        "role": "ADMIN"
      }
      """
    Then the response status should be 201
    And I save JSON path "id" as "userId"

    When I send a PUT request to "/api/users/${userId}" with body:
      """
      {
        "name": "Ravi Kumar",
        "role": "USER"
      }
      """
    Then the response status should be 200
    And the JSON at path "name" should be "Ravi Kumar"
    And the JSON at path "role" should be "USER"

    When I send a DELETE request to "/api/users/${userId}"
    Then the response status should be 204

  @patch_with_headers_and_query
  Scenario: PATCH with custom headers and query params
    When I send a PATCH request to "/api/orders/123/status" with headers:
      | X-Request-Id | 9f1a-abc-123 |
      | X-Source     | cucumber     |
    And I send a PATCH request to "/api/orders/123/status" with query params:
      | soft | true |
    Then the response status should be 200
    And the JSON at path "status" should exist

  @auth
  Scenario: Auth via explicit JWT and capture token
    Given I set the JWT token ""
    When I send a POST request to "/auth/login" with body:
      """
      { "username":"admin", "password":"admin@123" }
      """
    Then the response status should be 200
    And I save JSON path "accessToken" as "token"
    And I store "${token}" as "jwtToken"
    Given I set the JWT token "${jwtToken}"
    When I send a GET request to "/api/profile"
    Then the response status should be 200
    And the JSON at path "username" should be "admin"
```

> Tip: if you want simple **Scenario Outlines** for many endpoints, you can do:

```gherkin
Scenario Outline: Generic call
  When I send a <method> request to "<path>" with body:
    """
    <body>
    """
  Then the response status should be <status>

  Examples:
    | method | path            | body                         | status |
    | GET    | /api/health     |                              | 200    |
    | POST   | /api/ping       | {"msg":"hi"}                 | 201    |
    | DELETE | /api/temp/123   |                              | 204    |
```

---

# 7) Example JSON schema (optional)

```json
// src/test/resources/schemas/user-list.schema.json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["content","pageable"],
  "properties": {
    "content": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id","name","email"],
        "properties": {
          "id": { "type": ["integer", "string"] },
          "name": { "type": "string" },
          "email": { "type": "string", "format": "email" }
        }
      }
    },
    "pageable": {
      "type": "object",
      "required": ["pageNumber"],
      "properties": {
        "pageNumber": { "type": "integer" }
      }
    }
  }
}
```

---

# 8) Running the tests

* Local default: `mvn -q -DAPI_BASE_URL=http://localhost:8080 test`
* With JWT from env: `API_JWT=eyJ... mvn test`
* With system properties:
  `mvn test -Dapi.baseUrl=http://host.docker.internal:8080 -Dapi.jwt=eyJ...`

---

# 9) Useful extensions (drop-ins)

* **Random helpers**: add a tiny step to generate `${rand}`:

```java
@Given("I store a random suffix as {string}")
public void randomSuffix(String key) {
  String r = UUID.randomUUID().toString().substring(0,8);
  Hooks.CTX.get().putVar(key, r);
}
```

Use it in payloads like `email: "ravi.${rand}@example.com"`.

* **Array size checks**:

```java
@Then("the JSON array at path {string} should have size {int}")
public void arraySize(String path, int size) {
  List<?> arr = ctx().getLastResponse().jsonPath().getList(path);
  Assertions.assertThat(arr).hasSize(size);
}
```

* **Numeric comparisons**:

```java
@Then("the JSON number at path {string} should be > {double}")
public void jsonNumberGt(String path, double n) {
  Number v = ctx().getLastResponse().jsonPath().get(path);
  Assertions.assertThat(v.doubleValue()).isGreaterThan(n);
}
```

---

# 10) Testing a Spring Boot app without starting a server (optional)

If you prefer **MockMvc** over HTTP:

* Depend on `spring-boot-starter-test` (test scope) and wire a `MockMvc` bean in tests.
* Replace `Rest Assured` calls with `MockMvc` calls inside `ApiClient`. (Everything else—features, steps—stays the same.)
  Rest Assured also supports a **MockMvc module**, but the standalone approach above keeps the template portable across projects/services.

---

## That’s it

You now have a generic, extensible Cucumber harness that can hit **any** Spring REST endpoint with **any method**, reuse steps across services, and assert responses with JSONPath or schema validation. If you want, tell me your typical endpoints (e.g., `/api/expenses`, `/api/sip`, `/api/lumpsum`) and I’ll pre-author a feature file tailored to your project’s payloads and auth flow.
