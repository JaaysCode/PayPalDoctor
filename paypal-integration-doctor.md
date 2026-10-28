# PayPal Integration Doctor

> An AI agent that **unblocks developers integrating PayPal** by working against their *real sandbox*. It diagnoses errors, reproduces them, tests webhooks, scaffolds features and modernizes legacy code. It is delivered as an MCP server plus agent skills (Claude Code, Cursor, VS Code Copilot) and a hosted web console.

**Hackathon:** PayPal AI Hackathon (Devpost) · **Deadline:** Nov 12, 2026, 12:00 PM PST (3:00 PM Medellín)
**Stack:** Java 25 · Spring Boot 4.1 · Gradle (Kotlin DSL) · Spring AI (MCP server + ChatClient) · PostgreSQL/pgvector · Angular · Render

---

## 1. Problem & pitch

Integrating PayPal is easy when it works. When it doesn't, developers stall on:

- **Opaque errors:** `UNPROCESSABLE_ENTITY` with an `issue` like `ORDER_NOT_APPROVED` or `ITEM_TOTAL_MISMATCH`, or `invalid_client` because they used live credentials against the sandbox URL.
- **Webhooks:** events that never arrive, signature verification that keeps failing, not knowing which event types to subscribe to.
- **Outdated tutorials:** IPN, the v1 Payments API, NVP/SOAP and JS SDK v5 examples still dominate search results and community skills.
- **Context switching:** docs, the dashboard, Postman and the code editor are all separate places.

PayPal's existing AI tooling focuses on **migration** (VS Code assistant) and on **merchant operations** (the MCP server and Agent Toolkit for invoices, orders and so on). Nothing acts as a **debugging partner** that touches the developer's sandbox.

**One-liner:** *"Paste the error. The Doctor reproduces it in your sandbox, explains it, and fixes your code."*

### What makes it different (say this in the video)
| Existing tool | What it does | What the Doctor adds |
|---|---|---|
| PayPal VS Code AI assistant | Migrates NVP/SOAP → REST and JS v5 → v6 | Live sandbox debugging, error reproduction, webhook testing |
| PayPal MCP server / Agent Toolkit | Merchant actions (invoices, orders, refunds) | Developer-facing diagnostics: credential checks, error catalog, signature verification |
| Community PayPal skills | Static integration guides (often IPN / v5) | Current v6 / Orders v2 guidance + tools that *execute* and *verify* |

---

## 2. Features (MVP → stretch)

### MVP (must ship)
1. **Credential & environment check:** fetches an OAuth token and detects live-vs-sandbox mix-ups, missing scopes and expired apps.
2. **Error Doctor:** takes a PayPal error JSON or stack trace and returns the root cause, the fix, and a code patch.
   - Looks up the curated error catalog first (deterministic), then calls the LLM for context-specific advice.
   - Optionally **reproduces** the error in the sandbox using negative testing (mock response header).
3. **Checkout flow runner:** creates → (approval link) → captures an Orders v2 order step by step and shows each request and response.
4. **Webhook lab:**
   - Simulates any event type to the developer's URL (`POST /v1/notifications/simulate-event`).
   - Verifies a signature from the headers they paste (`POST /v1/notifications/verify-webhook-signature`).
   - Captures events in a built-in inspector endpoint.
5. **Agent skills pack:** five or six `SKILL.md` skills that teach any coding agent the *current* PayPal patterns and point it to the Doctor's tools.
6. **Web console (hosted):** a chat panel plus a live "sandbox timeline" of every API call the agent made. This is the hosted demo URL for the judges.

### Stretch
- **Legacy scanner:** scans a repo and flags IPN, `/v1/payments`, NVP/SOAP and the JS SDK v5 script URL, then proposes patches.
- **Subscription scaffolder:** from "monthly plan, 7-day trial, $9.99", it creates the product and plan in the sandbox and generates code.
- **Docs RAG:** answers grounded in PayPal docs, with citations (PayPal publishes `.md` pages and OpenAPI JSON on docs.paypal.ai).

---

## 3. Architecture

```
┌──────────────────────────┐      ┌────────────────────────────┐
│ Coding agents            │      │ Web console (Angular)      │
│ Claude Code / Cursor /   │      │ chat + sandbox timeline    │
│ VS Code Copilot          │      │ + webhook inspector        │
│ + /skills (SKILL.md)     │      └─────────────┬──────────────┘
└────────────┬─────────────┘                    │ REST + SSE
             │ MCP (Streamable HTTP)            │
┌────────────▼──────────────────────────────────▼─────────────┐
│                 PaypalAPP (Spring Boot 4.1, Java 25)        │
│                                                             │
│  MCP Server (spring-ai-starter-mcp-server-webmvc)  /mcp     │
│  Chat API (Spring AI ChatClient + same @Tool beans) /api    │
│  Webhook capture endpoint             /hooks/{sessionId}    │
│                                                             │
│  ┌──────────────┐ ┌───────────────┐ ┌─────────────────────┐ │
│  │ DoctorTools  │ │ ErrorCatalog  │ │ SessionStore        │ │
│  │ (@Tool)      │ │ (YAML rules)  │ │ (creds in memory,   │ │
│  └──────┬───────┘ └───────────────┘ │  TTL, never logged) │ │
│         │                           └─────────────────────┘ │
│  ┌──────▼─────────────────────┐   ┌───────────────────────┐ │
│  │ PayPalSandboxClient        │   │ RAG: pgvector +       │ │
│  │ (RestClient, SANDBOX ONLY) │   │ QuestionAnswerAdvisor │ │
│  └──────┬─────────────────────┘   └───────────────────────┘ │
└─────────┼───────────────────────────────────────────────────┘
          ▼
  https://api-m.sandbox.paypal.com   (OAuth, Orders v2, Webhooks,
                                      Catalog/Billing, Invoicing)
```

**Key decisions**
- **One set of tools, two front doors.** The same `@Tool` methods are exposed over MCP (for IDE agents) and used by the ChatClient (for the web console). There is no duplicated logic.
- **Sandbox-only by design.** The base URL is hard-coded to `api-m.sandbox.paypal.com`, and the server refuses anything that looks like live credentials. This is a security selling point, so mention it in the pitch.
- **Deterministic first, LLM second.** Known errors resolve from the catalog, which is fast, reliable and cheap. The LLM adds explanation and code patches tailored to the user's stack.
- **Raw `RestClient` instead of an SDK.** It gives one consistent code path, full control over headers (`PayPal-Request-Id`, `PayPal-Mock-Response`), and access to the raw error bodies the Doctor needs. PayPal's Java Server SDK (`com.paypal.sdk:paypal-server-sdk`) only covers part of the API surface, so we use it only if it saves time on Orders.
- **Run modes:**
  - *Local:* a developer runs the jar with `PAYPAL_CLIENT_ID` and `PAYPAL_CLIENT_SECRET` as environment variables. This is the safest mode for real code work.
  - *Hosted:* the console on Render; developers paste sandbox credentials into a short-lived session.

---

## 4. Project structure

Single Gradle module at the repo root (not a `doctor-server/` subfolder). `(exists)` marks what is already generated.

```
PaypalAPP/
├── LICENSE                      # MIT – required, visible in repo "About"
├── README.md                    # setup + run instructions for judges
├── Dockerfile
├── docker-compose.yml           # postgres+pgvector, app, web
├── build.gradle.kts             # (exists)
├── settings.gradle.kts          # (exists)  rootProject.name = "PaypalAPP"
├── gradlew, gradlew.bat, gradle/wrapper/   # (exists)
├── src/
│   ├── main/
│   │   ├── java/com/app/paypalapp/
│   │   │   ├── PaypalAppApplication.java     # (exists)
│   │   │   ├── config/AiConfig.java          # ChatClient, ToolCallbackProvider, RAG
│   │   │   ├── config/SecurityConfig.java    # open /mcp, /api, /hooks only
│   │   │   ├── paypal/PayPalSandboxClient.java
│   │   │   ├── paypal/ApiResult.java
│   │   │   ├── tools/CredentialTools.java
│   │   │   ├── tools/ErrorDoctorTools.java
│   │   │   ├── tools/OrderFlowTools.java
│   │   │   ├── tools/WebhookTools.java
│   │   │   ├── tools/ScaffoldTools.java      # stretch
│   │   │   ├── catalog/ErrorCatalog.java
│   │   │   ├── session/SessionStore.java
│   │   │   ├── web/ChatController.java       # SSE streaming
│   │   │   ├── web/HookCaptureController.java
│   │   │   └── web/TimelineController.java
│   │   └── resources/
│   │       ├── application.yaml              # (exists)
│   │       ├── error-catalog.yaml
│   │       └── prompts/system.st
│   └── test/java/com/app/paypalapp/
│       └── PaypalAppApplicationTests.java    # (exists)
├── web-console/                 # Angular app
└── skills/                      # Agent Skills pack
    ├── paypal-error-doctor/SKILL.md
    ├── paypal-checkout-v6/SKILL.md
    ├── paypal-orders-v2-server/SKILL.md
    ├── paypal-webhooks/SKILL.md
    ├── paypal-subscriptions/SKILL.md
    └── paypal-legacy-migration/SKILL.md
```

---

## 5. Building it with Spring Boot

### 5.1 Starting point (already in the repo)
The Gradle skeleton exists: **Java 25, Spring Boot 4.1.2, Gradle Kotlin DSL**, base package `com.app.paypalapp`, config in `src/main/resources/application.yaml`.

Already declared in `build.gradle.kts`: `starter-webmvc`, `starter-data-jpa`, `starter-security`, PostgreSQL driver, Lombok, JUnit. **Add:**
- Validation
- Actuator
- Spring AI *Model Context Protocol Server* (WebMVC)
- Spring AI *Anthropic Claude* (or OpenAI; any Spring AI chat model works)
- Jackson YAML (loads `error-catalog.yaml`)
- *PGvector* + vector-store advisor (RAG stretch only; add on stretch day, see 5.2)

Version notes for the Boot 4 line:
- Boot 4 needs **Spring AI 2.x**. Pin the release start.spring.io offers for Boot 4.1 on Day 1 and do not upgrade mid-hackathon. The `@Tool` approach below is the same API.
- Boot 4 ships **Jackson 3** (`tools.jackson.*` packages and group id). `JsonNode` usage in the snippets works the same; only imports change.
- The web starter is `spring-boot-starter-webmvc` (not `-web`).
- Lombok is available, but prefer `record` for DTOs (`ApiResult`, `Diagnosis`, `Credentials`).
- `starter-security` is on the classpath, so §5.11 is required or every endpoint returns 401.

### 5.2 `build.gradle.kts` (key parts)
```kotlin
plugins {
    java
    id("org.springframework.boot") version "4.1.2"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "com.app"
version = "0.0.1-SNAPSHOT"
description = "PaypalAPP"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}

repositories {
    mavenCentral()
}

// pin the Spring AI 2.x release that start.spring.io offers for Boot 4.1
extra["springAiVersion"] = "<spring-ai-2.x-version>"

dependencyManagement {
    imports {
        mavenBom("org.springframework.ai:spring-ai-bom:${property("springAiVersion")}")
    }
}

dependencies {
    // already present
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-webmvc")
    compileOnly("org.projectlombok:lombok")
    runtimeOnly("org.postgresql:postgresql")
    annotationProcessor("org.projectlombok:lombok")
    // ...test dependencies unchanged

    // add for the Doctor
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.ai:spring-ai-starter-mcp-server-webmvc")
    implementation("org.springframework.ai:spring-ai-starter-model-anthropic")
    implementation("tools.jackson.dataformat:jackson-dataformat-yaml")

    // stretch: RAG. Keep commented until you also add an embedding model
    // (Anthropic has none), otherwise PgVectorStore fails at startup.
    // implementation("org.springframework.ai:spring-ai-starter-vector-store-pgvector")
    // implementation("org.springframework.ai:spring-ai-advisors-vector-store")
}

tasks.withType<Test> {
    useJUnitPlatform()
}

tasks.bootJar {
    archiveFileName = "paypal-doctor.jar"   // stable name for Dockerfile / README
}
```

### 5.3 `application.yaml`
```yaml
server:
  port: ${PORT:8080}

spring:
  application:
    name: PaypalAPP
  # JPA + Postgres are on the classpath, so a datasource is mandatory at startup
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/ppdoctor}
    username: ${DB_USER:ppdoctor}
    password: ${DB_PASSWORD:ppdoctor}
  jpa:
    hibernate:
      ddl-auto: update
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: ${AI_MODEL}          # set the model name in env, not in code
    mcp:
      server:
        name: paypal-integration-doctor
        version: 0.1.0
        type: SYNC
        protocol: STREAMABLE           # endpoint: /mcp

management:
  endpoints:
    web:
      exposure:
        include: health

doctor:
  paypal:
    base-url: https://api-m.sandbox.paypal.com   # sandbox only – never configurable to live
  session-ttl: 2h
```
Local Postgres: `docker-compose.yml` runs `pgvector/pgvector:pg17` with db/user/password `ppdoctor`, matching the defaults above.

### 5.4 PayPal sandbox client (captures errors instead of throwing)
```java
@Component
public class PayPalSandboxClient {

  private static final String SANDBOX = "https://api-m.sandbox.paypal.com";
  private final RestClient http = RestClient.builder().baseUrl(SANDBOX).build();
  private final TimelineRecorder timeline;   // feeds the console's live timeline

  public PayPalSandboxClient(TimelineRecorder timeline) { this.timeline = timeline; }

  public String accessToken(Credentials c) {
    var form = new LinkedMultiValueMap<String, String>();
    form.add("grant_type", "client_credentials");
    ApiResult r = exchange(http.post().uri("/v1/oauth2/token")
        .headers(h -> h.setBasicAuth(c.clientId(), c.clientSecret()))
        .contentType(MediaType.APPLICATION_FORM_URLENCODED)
        .body(form), "POST /v1/oauth2/token");
    if (!r.ok()) throw new PayPalAuthException(r);   // Error Doctor explains this
    return r.json().get("access_token").asText();
  }

  public ApiResult post(Credentials c, String path, Object body, Map<String, String> extraHeaders) {
    String token = accessToken(c);                    // cache per session in real code
    return exchange(http.post().uri(path)
        .headers(h -> {
          h.setBearerAuth(token);
          h.set("PayPal-Request-Id", UUID.randomUUID().toString());
          extraHeaders.forEach(h::set);               // e.g. PayPal-Mock-Response
        })
        .contentType(MediaType.APPLICATION_JSON)
        .body(body), "POST " + path);
  }

  private ApiResult exchange(RestClient.RequestBodySpec spec, String label) {
    ApiResult result = spec.exchange((req, res) -> new ApiResult(
        res.getStatusCode().value(),
        res.bodyTo(String.class),
        res.getHeaders().getFirst("Paypal-Debug-Id")));
    timeline.record(label, result);
    return result;
  }
}
```

### 5.5 Tools (exposed over MCP and used by the chat)
```java
@Component
public class ErrorDoctorTools {

  private final ErrorCatalog catalog;
  private final PayPalSandboxClient paypal;
  private final SessionStore sessions;

  // constructor omitted

  @Tool(description = """
      Diagnose a PayPal REST API error. Input: the raw error JSON or log excerpt.
      Returns the error name, the issue code, the likely root cause, the fix,
      and the PayPal debug_id if present.""")
  public Diagnosis diagnoseError(@ToolParam(description = "Raw PayPal error JSON or log text")
                                 String rawError) {
    ParsedError e = ErrorParser.parse(rawError);   // name, details[].issue, debug_id
    return catalog.lookup(e)
        .map(rule -> Diagnosis.fromRule(e, rule))
        .orElse(Diagnosis.unknown(e));             // LLM fills in reasoning when unknown
  }

  @Tool(description = """
      Reproduce a PayPal Orders v2 error in the developer's SANDBOX using negative
      testing, so they can see the exact response their code must handle.""")
  public ApiResult reproduceOrderError(String sessionId,
      @ToolParam(description = "Issue code, e.g. INSTRUMENT_DECLINED") String issue) {
    var creds = sessions.credentials(sessionId);
    var mock = Map.of("PayPal-Mock-Response",
        "{\"mock_application_codes\":\"" + issue + "\"}");
    return paypal.post(creds, "/v2/checkout/orders", SampleOrders.basicUsd(), mock);
  }
}
```

```java
@Component
public class WebhookTools {

  @Tool(description = "Send a simulated PayPal webhook event of the given type to a URL (sandbox).")
  public ApiResult simulateWebhook(String sessionId, String url, String eventType) {
    return paypal.post(sessions.credentials(sessionId), "/v1/notifications/simulate-event",
        Map.of("url", url, "event_type", eventType, "resource_version", "1.0"), Map.of());
  }

  @Tool(description = """
      Verify a received webhook's signature with PayPal. Needs the PAYPAL-* transmission
      headers, the raw body and the webhook_id. Explains why verification fails.""")
  public SignatureCheck verifySignature(String sessionId, Map<String, String> headers,
                                        String rawBody, String webhookId) { /* ... */ }
}
```

### 5.6 Wiring tools + chat
```java
@Configuration
public class AiConfig {

  @Bean
  ToolCallbackProvider doctorTools(CredentialTools a, ErrorDoctorTools b,
                                   OrderFlowTools c, WebhookTools d) {
    return MethodToolCallbackProvider.builder().toolObjects(a, b, c, d).build();
  }

  @Bean
  ChatClient doctorChat(ChatClient.Builder builder, ToolCallbackProvider tools,
                        @Value("classpath:prompts/system.st") Resource system) {
    return builder.defaultSystem(system)
                  .defaultToolCallbacks(tools)
                  .build();
  }
}
```
The `ToolCallbackProvider` bean is what the MCP server auto-configuration exports, and the ChatClient reuses the same bean. The chat endpoint streams over SSE with `chatClient.prompt().user(msg).stream().content()`.

**System prompt essentials (`system.st`):**
- You are a PayPal integration debugger. Use **sandbox only**.
- Always call a tool to *verify* before claiming a cause.
- Prefer current APIs: Orders v2, JS SDK v6, webhooks over IPN.
- Show the `debug_id` when present and tell the user to include it in support tickets.
- Never echo secrets.

### 5.7 Error catalog (deterministic layer)
```yaml
# error-catalog.yaml
- match: { name: "invalid_client" }
  cause: "Client ID/secret rejected — usually LIVE credentials used against the SANDBOX endpoint (or vice versa), or a typo/whitespace."
  fix: "Copy credentials from Dashboard → Apps & Credentials with the Sandbox toggle on; confirm base URL is api-m.sandbox.paypal.com."
- match: { name: "UNPROCESSABLE_ENTITY", issue: "ORDER_NOT_APPROVED" }
  cause: "Capture called before the buyer approved the order."
  fix: "Capture only in onApprove / after redirect back with the approved order ID."
- match: { name: "UNPROCESSABLE_ENTITY", issue: "ORDER_ALREADY_CAPTURED" }
  cause: "Capture called twice (double click, retry without idempotency)."
  fix: "Send a stable PayPal-Request-Id per capture; disable the button after first click."
- match: { name: "UNPROCESSABLE_ENTITY", issue: "INSTRUMENT_DECLINED" }
  cause: "Funding source declined."
  fix: "Call actions.restart() (JS SDK) so the buyer can pick another funding source."
- match: { name: "UNPROCESSABLE_ENTITY", issue: "ITEM_TOTAL_MISMATCH" }
  cause: "amount.breakdown.item_total ≠ sum(unit_amount × quantity)."
  fix: "Recompute totals server-side; watch rounding and currency decimals."
# … aim for 25–40 entries; this is the "knowledge" judges will see is real
```
Before the demo, check every issue code against the Orders v2 error reference and the negative-testing docs. Only keep codes you have actually reproduced in the sandbox.

### 5.8 Skills pack (example)
```markdown
---
name: paypal-error-doctor
description: Use when a PayPal REST API call fails (4xx/5xx, UNPROCESSABLE_ENTITY, invalid_client, webhook signature failures) to diagnose the cause and patch the code.
---
# PayPal Error Doctor

1. Collect the raw error body, HTTP status and `Paypal-Debug-Id` header from the user's logs.
2. Call the `diagnoseError` tool (paypal-integration-doctor MCP server) with the raw body.
3. If the cause is not certain, call `reproduceOrderError` with the issue code to confirm in sandbox.
4. Patch the user's code. Rules:
   - Server-side create/capture with Orders v2 (`/v2/checkout/orders`), never `/v1/payments`.
   - Idempotency: send `PayPal-Request-Id` on POSTs that create or capture.
   - Never put secrets in frontend code.
5. Report: cause → fix → diff → debug_id.
```
Other skills follow the same pattern: when to use, which Doctor tools to call, and the *current* patterns plus anti-patterns.

**Connecting a coding agent (local run):**
```bash
PAYPAL_CLIENT_ID=... PAYPAL_CLIENT_SECRET=... ANTHROPIC_API_KEY=... AI_MODEL=... \
  ./gradlew bootRun

claude mcp add --transport http paypal-doctor http://localhost:8080/mcp
cp -r skills/* .claude/skills/
```

### 5.9 Web console (Angular)
- **Left:** chat (SSE stream).
- **Right:** **Sandbox Timeline** that lists each API call with method, path, status, `debug_id` and an expandable body.
- **Webhook Inspector tab:** shows a unique capture URL (`/hooks/{sessionId}`) and the events received, with signature verification status.
- **Credentials modal:** sandbox only, kept in memory for the session TTL, with a "wipe session" button.

### 5.10 Deploy (Render)
```dockerfile
# Dockerfile (repo root)
FROM eclipse-temurin:25-jdk AS build
WORKDIR /app
COPY . .
RUN ./gradlew --no-daemon -q bootJar -x test

FROM eclipse-temurin:25-jre
COPY --from=build /app/build/libs/paypal-doctor.jar /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```
- Render web service from the Dockerfile, plus Render Postgres (enable `vector`) and a static site for the Angular build.
- Render env vars: `ANTHROPIC_API_KEY`, `AI_MODEL`, `DB_URL`, `DB_USER`, `DB_PASSWORD`. Render gives the `jdbc:` URL only after you convert its `postgres://` connection string, so build `DB_URL` by hand.
- Render is a hackathon sponsor, so mention it in the submission (it has its own prizes).

### 5.11 Spring Security config
`spring-boot-starter-security` is already in `build.gradle.kts`. Left alone, it puts every endpoint behind a generated password and CSRF, so MCP clients and PayPal's webhook POSTs get 401/403. Open only what the Doctor serves:
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

  @Bean
  SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.ignoringRequestMatchers("/mcp/**", "/api/**", "/hooks/**"))
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(a -> a
            .requestMatchers("/mcp/**", "/api/**", "/hooks/**", "/actuator/health").permitAll()
            .anyRequest().denyAll())
        .build();
  }
}
```
- `/hooks/**` must be public because PayPal posts to it; webhook signature verification is its authentication.
- `/mcp` and `/api` only hold a `sessionId`, never a secret, but add per-session rate limiting before the hosted deploy (see §6).

---

## 6. Security (our edge, so show it)
- Sandbox base URL is hard-coded; a credential check rejects anything that authenticates only against live.
- Secrets are never logged, never sent to the LLM (tools receive a `sessionId`, not the secret), and expire with the session.
- Prompt-injection guard: webhook bodies and API responses are treated as data and passed to the model as quoted tool output, never as instructions.
- Rate-limit the hosted console per session.

---

## 7. Plan: Oct 29 → Nov 12

| Day | Date | Goal |
|---|---|---|
| 1 | Wed Oct 29 | Repo + LICENSE, sandbox app + test accounts, add Spring AI deps to the existing Gradle skeleton, MCP `/mcp` responding (test with MCP Inspector) |
| 2 | Thu Oct 30 | `PayPalSandboxClient` + timeline recorder; `checkCredentials` tool |
| 3 | Fri Oct 31 | Error parser + catalog (first 15 entries); `diagnoseError` |
| 4 | Sat Nov 1 | Negative testing → `reproduceOrderError`; verify which issue codes reproduce |
| 5 | Sun Nov 2 | Webhook tools: simulate, verify signature, capture endpoint |
| 6 | Mon Nov 3 | ChatClient + SSE chat endpoint; system prompt; end-to-end chat test |
| 7 | Tue Nov 4 | Angular console: chat + timeline |
| 8 | Wed Nov 5 | Webhook inspector UI; credentials/session flow |
| 9 | Thu Nov 6 | Skills pack (6 skills); test in Claude Code + Cursor against local server |
| 10 | Fri Nov 7 | Deploy to Render; README with run instructions; catalog → 30+ entries |
| 11 | Sat Nov 8 | Stretch: legacy scanner **or** subscription scaffolder (pick one) |
| 12 | Sun Nov 9 | Bug bash with a "broken demo app" (see §8); polish |
| 13 | Mon Nov 10 | **Code freeze.** Record demo video takes |
| 14 | Tue Nov 11 | Edit video (<3 min), write Devpost text, screenshots |
| 15 | Wed Nov 12 | Submit **before noon Medellín** (deadline 3:00 PM); final checks |

**Suggested split (3 people):**
- **A:** backend tools and the PayPal client.
- **B:** AI layer (prompts, catalog, skills, RAG).
- **C:** Angular console, deploy, video.

With two people, merge B into A and C.

---

## 8. Demo video script (≤ 3:00)
1. **0:00–0:20 Hook.** "Every PayPal developer has lost an afternoon to `UNPROCESSABLE_ENTITY`." Show a broken sample checkout app.
2. **0:20–1:10 Error Doctor in Claude Code.** Paste the error. The agent calls `diagnoseError`, reproduces it in the sandbox, explains it, and patches the code. Re-run: payment captured ✅.
3. **1:10–1:50 Webhook lab.** "My webhook never verifies." Doctor simulates `PAYMENT.CAPTURE.COMPLETED`, captures it, finds the bug (re-serialized body instead of the raw body), fixes it, and verification passes ✅.
4. **1:50–2:30 Web console.** Same flow for a developer without an IDE agent, with the live sandbox timeline and `debug_id`s visible.
5. **2:30–3:00 Why it matters.** Sandbox-only safety, works in any MCP client, skills are open source. Who it's for: every developer integrating PayPal.

Build a deliberately **broken demo app** (a small Spring Boot or Node checkout with 3–4 planted bugs) in the repo, so judges can reproduce the demo themselves.

---

## 9. Submission checklist
- [ ] Public GitHub repo, **LICENSE visible in About**, all source + assets
- [ ] README: what it is, architecture diagram, **step-by-step run instructions**, env vars, how to connect Claude Code/Cursor
- [ ] Hosted console URL (Render) working with fresh sandbox credentials
- [ ] YouTube video < 3 min, public, shows it working end-to-end, no copyrighted music
- [ ] "Tools used" section: AI model + Spring AI (ChatClient, MCP server, tool calling), what the LLM does vs. the deterministic catalog
- [ ] PayPal technologies listed: OAuth, Orders v2, Webhooks (simulate + verify), negative testing, (Catalog/Billing if stretch)
- [ ] Devpost text: problem → who → solution → how it's built → what's next

---

## 10. Risks & mitigations
| Risk | Mitigation |
|---|---|
| Scope creep | MVP list in §2 is the contract; stretch only after Day 10 |
| Negative-testing codes not all reproducible | Verify on Day 4; demo only codes that reproduce |
| Judges see it as "another docs chatbot" | Every demo beat must show a **real sandbox call** in the timeline |
| Spring AI API differences between versions | Pin one stable version on Day 1; don't upgrade mid-hackathon |
| LLM hallucinating causes | Catalog-first; system prompt forces tool verification before conclusions |
| Hosted demo abuse / secret leakage | Sandbox-only, session TTL, secrets never sent to the model |

---

## References
- Hackathon: https://paypalaihackathon.devpost.com/
- PayPal REST APIs: https://developer.paypal.com/api/rest/
- PayPal AI docs & MCP quickstart: https://docs.paypal.ai/developer/tools/ai/mcp-quickstart
- Simulate webhook event: https://docs.paypal.ai/reference/api/rest/simulate-event/simulate-webhook-event
- PayPal Java Server SDK (optional): https://github.com/paypal/PayPal-Java-Server-SDK
- Spring AI MCP server starter: https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html
