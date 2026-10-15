# JEV / System One support in AgentScope Java

This is a short, read-only overview of JEV (TypeSafe [System One](https://docs.typesafe.ai/))
support in this repository, grounded in the source. For the full user guide see
[`v2/en/jev/index`](v2/en/jev/index.md) (EN) and [`v2/zh/jev/index`](v2/zh/jev/index.md) (ZH).

Jev is **not** a chat model and is intentionally **not** registered as an AgentScope `Model`
provider. Use it when application code needs a fast, typed, calibrated decision such as routing,
scoring, or classification.

## 1. Modules

| Module | Path | Role |
| --- | --- | --- |
| `agentscope-extensions-jev` | `agentscope-extensions/agentscope-extensions-judge/agentscope-extensions-jev/` | TypeSafe System One HTTP client, request/result DTOs, retry, validation |
| `agentscope-jev-spring-boot-starter` | `agentscope-extensions/agentscope-spring-boot-starters/agentscope-jev-spring-boot-starter/` | Spring Boot auto-configuration and `@ConfigurationProperties` |
| BOM / distribution | `agentscope-distribution/agentscope-bom/pom.xml` | Managed dependency versions |

The JEV extension is grouped under the `agentscope-extensions-judge` parent module.

## 2. Key classes

Package `io.agentscope.extensions.judge.jev`:

- `JevClient` — Java client for the System One endpoint (`POST /v1/systemone`). Reactive
  `Mono<SystemOneResult>` plus a `systemOneBlocking` convenience. Defaults: base URL
  `https://api.typesafe.ai`, model `jev-latest`, 5s per-attempt timeout.
- `JevRetryPolicy` — `record(int maxRetries, Duration initialBackoff)`; defaults
  `(2, Duration.ofMillis(500))`. Retries HTTP `429`, `529`, and `5xx` with exponential backoff.
- `JevException` — thrown for non-retryable client errors, exhausted retries, and invalid
  responses; carries status code, response body, and a `retryable` flag.
- `SystemOneRequest` / `SystemOneResult` — request/result records. A request carries `state`
  and a map of named `Question`s; a result carries `model`, `answers`, and `Usage`.
- `Question` sealed hierarchy: `NoulQuestion`, `ChoiceQuestion`, `ScoreQuestion`.
- `Answer` sealed hierarchy: `NoulAnswer`, `ChoiceAnswer`, `ScoreAnswer`.
- `Usage` — token/usage accounting returned by System One.

Reference middlewares in `io.agentscope.extensions.judge.jev.example` (attachable via
`ReActAgent.builder().middleware(...)` or copy-and-tune):

| Middleware | Interception point | Purpose |
| --- | --- | --- |
| `JevToolSelectionMiddleware` | `onReasoning` | Filter/rank tools sent to the primary model |
| `JevModelRouterMiddleware` | `onAgent` / `onModelCall` | Route between configured models |
| `JevAutoModeMiddleware` | `onActing` | Assess risk before tool execution |

Spring Boot starter, package `io.agentscope.spring.boot.jev`:

- `JevAutoConfiguration` — `@AutoConfiguration` registered via
  `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`; creates a
  `JevClient` bean unless one already exists, gated by `agentscope.jev.enabled` (default `true`).
- `JevProperties` — `@ConfigurationProperties(prefix = "agentscope.jev")` with `enabled`,
  `apiKey`, `baseUrl`, `model`, `timeout`, and nested `retry` (`maxRetries`, `initialBackoff`).
- `JevClientBuilderCustomizer` — hook for advanced builder tweaks.

## 3. Maven coordinates

Project version is driven by the `${revision}` property (currently `2.0.4-SNAPSHOT` in the root
POM). Import the BOM to keep versions aligned:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.agentscope</groupId>
      <artifactId>agentscope-bom</artifactId>
      <version>${agentscope.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

Client only:

```xml
<dependency>
  <groupId>io.agentscope</groupId>
  <artifactId>agentscope-extensions-jev</artifactId>
  <version>${agentscope.version}</version>
</dependency>
```

Spring Boot starter:

```xml
<dependency>
  <groupId>io.agentscope</groupId>
  <artifactId>agentscope-jev-spring-boot-starter</artifactId>
  <version>${agentscope.version}</version>
</dependency>
```

Both artifacts are listed in `agentscope-distribution/agentscope-bom/pom.xml`.

## 4. How to enable

### Plain Java

Build a `JevClient` and call `systemOne` / `systemOneBlocking`:

```java
JevClient client =
        JevClient.builder()
                .apiKey(System.getenv("TYPESAFE_API_KEY")) // or JEV_API_KEY fallback
                .baseUrl("https://api.typesafe.ai")
                .model("jev-latest")
                .retryPolicy(new JevRetryPolicy(2, Duration.ofMillis(500)))
                .timeout(Duration.ofSeconds(5))
                .build();

SystemOneRequest request =
        SystemOneRequest.builder()
                .state("My payouts have been failing for 3 days.")
                .question(
                        "team",
                        new ChoiceQuestion(
                                "Which team should handle this?",
                                Map.of(
                                        "billing", "Payments and refunds",
                                        "technical", "Bugs and integrations")))
                .build();

SystemOneResult result = client.systemOneBlocking(request);
ChoiceAnswer answer = (ChoiceAnswer) result.answers().get("team");
```

API key resolution: builder value → `TYPESAFE_API_KEY` env var → `JEV_API_KEY` env var.

### Spring Boot

Add the starter and configure under `agentscope.jev`:

```yaml
agentscope:
  jev:
    enabled: true                 # default; set false to disable auto-config
    api-key: ${TYPESAFE_API_KEY:}  # optional; falls back to TYPESAFE_API_KEY / JEV_API_KEY
    base-url: https://api.typesafe.ai
    model: jev-latest
    timeout: 5s
    retry:
      max-retries: 2
      initial-backoff: 500ms
```

For advanced configuration, register a `JevClientBuilderCustomizer` bean.

### As agent middleware

Attach a reference middleware to a `ReActAgent`:

```java
ReActAgent agent =
        ReActAgent.builder()
                .name("assistant")
                .model(model)
                .toolkit(toolkit)
                .middleware(
                        JevToolSelectionMiddleware.builder(client)
                                .alwaysIncludeTools(Set.of("load_skill_through_path", "reset_tools"))
                                .maxTools(3)
                                .confidenceThreshold(0.5)
                                .failOpen(true)
                                .build())
                .build();
```

`JevModelRouterMiddleware` and `JevAutoModeMiddleware` follow the same builder pattern; see the
full guide for their options and interception semantics.
