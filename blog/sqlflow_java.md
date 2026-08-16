title: Introducing SqlFlow: Building Resilient Workflows with SqlFlow (Java SDK)
date: 2026-08-16 11:31
tags: java
category: java
slug: sqlflow_java
author: Philipp Wagner
summary: This article shows how to build an Autonomous AI Agent with SqlFlow and Java.

SqlFlow is a simple durable execution workflow system for PostgreSQL and SQL Server. It handles scheduling 
and retries, without needing any other services to run in addition to PostgreSQL or SQL Server.

The SQL Scripts for the SqlFlow Database Schemas are available here:

* [https://github.com/bytefish/SqlFlow/blob/main/sql/](https://github.com/bytefish/SqlFlow/blob/main/sql)

It took a large deal of inspiration from [Absurd], but features a simpler database model. 

[Absurd]: https://github.com/earendil-works/absurd

## Table of contents ##

[TOC]

## Getting Started with the Database ##

You'll start by creating the `ssf` database schema and tables:

* PostgreSQL: `sql/ssf-postgres.sql`
* SQL Server: `sql/ssf-sqlserver.sql`

## Getting Started with the Java SDK ##

The SqlFlow SDK for Java is available in Maven Central repository:

```xml
<dependencies>
    <!-- SqlFlow Core Module -->
    <dependency>
        <groupId>de.bytefish.sqlflow</groupId>
        <artifactId>sqlflow-core</artifactId>
        <version>1.0.1</version>
    </dependency>

    <!-- SqlFlow PostgreSQL Module -->
    <dependency>
        <groupId>de.bytefish.sqlflow</groupId>
        <artifactId>sqlflow-postgres</artifactId>
        <version>1.0.1</version>
    </dependency>

    <!-- SqlFlow SQL Server Module -->
    <dependency>
        <groupId>de.bytefish.sqlflow</groupId>
        <artifactId>sqlflow-sqlserver</artifactId>
        <version>1.0.1</version>
    </dependency>
    
</dependencies>
```

Use either the `sqlflow-postgres` or `sqlflow-sqlserver` module.

## Java SDK: Building a Durable AI Agent ##

### What we are going to build ###

The classic examples for durable execution are usually e-commerce checkouts or payment processing scenarios. But there's another rapidly 
growing use case developers are dealing with: Autonomous AI Agents. Building AI agents that interact with external APIs, write code, 
or execute complex workflows introduces challenges.

1. LLM API calls are inherently slow, prone to timeouts or rate limits. And they are also quite expensive, right? If a server crashes 
or restarts while waiting for a 30-second AI generation, standard async and await state is lost forever. 
2. You don't want an AI to push code to production or execute financial transactions without a human looking at it. Agents need to pause 
their execution, ask a human for permission and resume only when approved. This is sometimes hours or days later.

Traditional approaches require you to build complex state machines, database polling loops, or heavy external infrastructure. With SqlFlow, 
we can write our agent as standard, sequential C# code. The framework will automatically checkpoint the state to Postgres, sleep without 
blocking server threads, and wake up exactly where it left off.

### Building an Agent Job ###

To demonstrate how durable execution with SqlFlow works, we are going to build an autonomous AI agent that 
fixes bugs. The workflow is quickly laid out as: 

1. The agent receives a GitHub issue ID and fetches the stack trace.  
2. It generates a potential code fix using a Large Language Model (LLM).  
3. It pauses and asks a human for approval.  
4. If the human rejects the fix and provides feedback, the agent tries again (up to 3 times).  
5. If approved, it creates a Pull Request. If it fails 3 times, it escalates to a senior developer.

So first, let's define the data models that represent our inputs, states and final output:

```java
public record AgentTask(
        @JsonProperty("issue_id") String issueId
) {}

public record Issue(
        @JsonProperty("stack_trace") String stackTrace
) {}

public record Solution(
        @JsonProperty("patched_code") String patchedCode
) {}

public record HumanApproval(
        @JsonProperty("approved") boolean approved,
        @JsonProperty("reason") String reason
) {}

public record AgentResult(
        @JsonProperty("success") boolean success,
        @JsonProperty("pull_request_url") String pullRequestUrl,
        @JsonProperty("reason") String reason
) {}
```

### The LLM Service ###

Next, we need a service to handle the AI code generation. In the real world, calling an LLM is a slow (and expensive) and the HTTP 
requests might fail or time out. We are wrapping these expensive calls with SqlFlow, so we don't lose all our state, if the 
server crashes.

For this demonstration, we are simulating ab LLM API call with some delay and return a hardcoded "code fixes" based on a reviewer's feedback:

```java
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

public interface LlmService {
    Solution generateFix(String log, String lastFeedback);
}

@Service
public class DefaultLlmService implements LlmService {
    private static final Logger logger = LoggerFactory.getLogger(DefaultLlmService.class);

    @Override
    public Solution generateFix(String log, String lastFeedback) {
        logger.info("Agent is thinking: 'Learned from feedback: {}'", lastFeedback);

        // Simulate expensive LLM call
        ServiceUtils.simulateDelay(2500);

        String code = lastFeedback.contains("error handling")
                ? "// AI: Improved Logging & Error-Handling added\nif(data == null) throw new IllegalArgumentException();"
                : "// AI: Simple Fix for the NullReferenceException\nif(data == null) return;";

        logger.info("LLM has generated a potential fix: \n{}", code);

        return new Solution(code);
    }
}
```

The agent needs to interact with the outside world. The GitHub service handles fetching the initial issue details and creating the final 
Pull Request. Whenever the LLM has generated has generated a solution, a human review is requested. If the LLM has been using more than 
a maximum amounts, the issue is escalated to a lead developer.

```java
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

public interface GitHubService {
    Issue getIssueDetails(String id);

    String createPullRequest(String id, String code);

    void requestHumanReview(String issueId, Solution proposedFix, String correlationId);

    void escalateToSenior(String id, String reason);
}

public class DefaultGitHubService implements GitHubService {
    private static final Logger logger = LoggerFactory.getLogger(DefaultGitHubService.class);

    @Override
    public Issue getIssueDetails(String issueId) {
        logger.info("GitHub: Gets Ticket #{} details from the Repository...", issueId);
        ServiceUtils.simulateDelay(800);
        return new Issue("NullReferenceException at PaymentGateway.java:42");
    }

    @Override
    public String createPullRequest(String issueId, String code) {
        logger.info("GitHub: PR for Issue #{} has been created...", issueId);
        ServiceUtils.simulateDelay(1200);
        return "https://github.com/company/repo/pull/" + (int) (Math.random() * 9000 + 1000);
    }

    @Override
    public void escalateToSenior(String id, String reason) {
        logger.error("ESCALATION to Senior Developer: Issue #{} - Reason: {}", id, reason);
        ServiceUtils.simulateDelay(500);
    }

    @Override
    public void requestHumanReview(String issueId, Solution proposedFix, String correlationId) {
        logger.info("ACTION REQUIRED: Solution for Issue #{} with Correlation-ID {} has been created: {}...",
                issueId, correlationId, proposedFix.patchedCode());
        ServiceUtils.simulateDelay(1200);
    }
}
```

### The Autonomous Agent Job ###

We define our logic inside an `IJob`. The magic is in the `ctx.Step` method: every time a step completes, its result is automatically checkpointed to the Postgres 
database. If the process crashes or is restarted, the framework replays the job. It skips the already completed steps and loads their results directly from 
the database.

And then instead of blocking a thread with `Task.Delay` or an infinite polling loop, we use `ctx.AwaitEvent` to wait for human interaction. This instructs the 
engine to safely suspend the workflow state to the database and free up the worker until an external system fires the specific event being awaited.

```java
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

@Component
public class AutonomousAgentJob implements Job<AgentTask, AgentResult> {

    private static final Logger logger = LoggerFactory.getLogger(AutonomousAgentJob.class);

    private final LlmService llmService;
    private final GitHubService gitHubService;
    private final LocalNotificationService localNotificationService;

    public AutonomousAgentJob(LlmService llmService, GitHubService gitHubService, LocalNotificationService localNotificationService) {
        this.llmService = llmService;
        this.gitHubService = gitHubService;
        this.localNotificationService = localNotificationService;
    }

    @Override
    public AgentResult execute(TaskContext ctx, AgentTask task) throws Exception {
        logger.info("Agent starts researching ticket {}", task.issueId());

        Issue bugReport = ctx.step("fetch-issue-context", Issue.class, () ->
                gitHubService.getIssueDetails(task.issueId())
        );

        boolean isApproved = false;
        int attempt = 0;
        String lastFeedback = "Initial Attempt";

        while (!isApproved && attempt < 3) {
            attempt++;
            String correlationId = ctx.getTaskId() + "-attempt-" + attempt;

            logger.info("Attempt {}/3: Generating a fix based on: {}", attempt, lastFeedback);

            final String currentFeedback = lastFeedback; // Für Lambda effectively final machen

            Solution proposedFix = ctx.step("generate-code-fix-" + attempt, Solution.class, () ->
                    llmService.generateFix(bugReport.stackTrace(), currentFeedback)
            );

            ctx.step("notify-reviewer-" + attempt, () -> {
                gitHubService.requestHumanReview(task.issueId(), proposedFix, correlationId);

                localNotificationService.notifyReviewer(task.issueId(), correlationId);
            });

            logger.info("Review for {} has been requested. Agent goes idle and waits for the code review...", correlationId);

            Optional<JsonNode> reviewOpt = ctx.awaitEvent(
                    "agent-approval:" + task.issueId() + ":" + correlationId,
                    "wait-for-human-review-" + attempt,
                    null,
                    JsonNode.class
            );

            if (reviewOpt.isPresent()) {
                JsonNode review = reviewOpt.get();
                isApproved = review.has("approved") && review.get("approved").asBoolean();
                lastFeedback = review.has("reason") ? review.get("reason").asText() : "No feedback has been given";

                if (!isApproved) {
                    logger.warn("Attempt {} has been rejected: {}", attempt, lastFeedback);
                }
            }
        }

        if (isApproved) {
            logger.info("Fix approved. Creating Pull Request...");

            String prUrl = ctx.step("create-pull-request", String.class, () ->
                    gitHubService.createPullRequest(task.issueId(), "apply-fix")
            );

            logger.info("Mission accomplished, the PR has been created: {}", prUrl);
            return new AgentResult(true, prUrl, null);

        } else {
            logger.error("Maximum number of attempts reached. Escalates ticket {} to a human.", task.issueId());

            ctx.step("notify-senior-developer", () -> {
                gitHubService.escalateToSenior(task.issueId(), "Agent didn't find a solution after 3 attempts.");
            });

            return new AgentResult(false, null, "Escalated to human supervisor after 3 failures.");
        }
    }
}
```

### Interacting with the System: Providing a Controller with HTTP Endpoints ###

Now there are two HTTP endpoints for interacting with the Job: 

* The `/agent/start` endpoint kicks off the process asynchronously and returns immediately.
* The `/agent/review/...` endpoint acts as our callback webhook. 
    * When the human reviewer approves or rejects a fix, this endpoint emits the event back into the queue, which then wakes up the sleeping job right where it left off.

Let's define an `AgentController` for it.

```java
@RestController
@RequestMapping("/agent")
public class AgentController {

    private final ISqlFlow sqlFlow;

    public AgentController(ISqlFlow sqlFlow) {
        this.sqlFlow = sqlFlow;
    }

    @PostMapping("/start")
    public Map<String, String> startAgent(
            @RequestBody AgentTask task) {

        SpawnResult result = sqlFlow.spawn(new SpawnOptions("ai-agent-queue", null, null, null), "solve-bug", task);

        return Map.of(
                "runId", result.runId(),
                "taskId", result.taskId(),
                "status", "Agent dispatched to fix Issue #" + task.issueId());
    }

    @PostMapping("/review/{issueId}/{correlationId}")
    public Map<String, String> review(
            @PathVariable("issueId") String issueId,
            @PathVariable("correlationId") String correlationId,
            @RequestBody HumanApproval approval) {

        String eventName = "agent-approval:" + issueId + ":" + correlationId;

        sqlFlow.emitEvent(
                new EmitEventOptions("ai-agent-queue"),
                eventName,
                approval);

        String message = approval.approved()
                ? "Fix for " + correlationId + " approved. Agent is now completing its work."
                : "Fix for " + correlationId + " rejected. Agent tries again with feedback: '" + approval.reason() + "'";

        return Map.of("message", message);
    }
}
```

### Putting It All Together: Dependency Injection ###

We are using Spring Boot, so we are using a class annotated with `@Configuration` to configure our dependencies. We 
are using TestContainers to spin up a new Postgres instance.

```java
@Configuration
public class SqlFlowConfiguration {

    @Bean(destroyMethod = "stop")
    public PostgreSQLContainer<?> postgresContainer() {
        PostgreSQLContainer<?> postgres =
                new PostgreSQLContainer<>("postgres:18")
                        .withInitScript("ssf-postgres.sql");

        postgres.start();
        return postgres;
    }

    @Bean
    public DataSource dataSource(
            PostgreSQLContainer<?> postgresContainer) {

        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(postgresContainer.getJdbcUrl());
        config.setUsername(postgresContainer.getUsername());
        config.setPassword(postgresContainer.getPassword());
        config.setMaximumPoolSize(10);

        return new HikariDataSource(config);
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule());
    }

    @Bean
    public JobFactory springJobFactory(ApplicationContext context) {
        return context::getBean;
    }

    @Bean
    public ISqlFlow sqlFlow(
            DataSource dataSource,
            ObjectMapper mapper,
            JobFactory jobFactory) {

        PostgresFlowDatabase db = new PostgresFlowDatabase(dataSource, mapper);

        ISqlFlow client = new SqlFlow(db, mapper);

        client.createQueue("ai-agent-queue");
        client.useJob(
                jobFactory,
                mapper,
                "solve-bug",
                3,
                AutonomousAgentJob.class,
                AgentTask.class);

        return client;
    }

    @Bean
    public SqlFlowWorker sqlFlowWorker(ISqlFlow client) {
        SqlFlowWorker worker = new SqlFlowWorker(
                WorkerOptions.builder()
                        .workerId("spring-worker-1")
                        .queue("ai-agent-queue")
                        .pollInterval(1.0)
                        .concurrency(1)
                        .build(),
                client);

        Thread.ofVirtual().start(worker);
        return worker;
    }
}
```

Spring Boot now auto-wires everything, and what's left is to define the application entry point.

```java
@SpringBootApplication
public class AiSampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(AiSampleApplication.class, args);
    }
}
```

## An Example Session with the AI Agent Job ##

### Getting the Tooling right ###

It's not stone age. I want to use tooling to fire my HTTP requests. There's somewhat of a standard 
established for tooling, which is the `http` format for HTTP requests.

And while it's easy to use `*.http` files with Visual Studio, IntelliJ doesn't come with a UI 
in it's Community Edition. But do not fear, you don't have to fight `curl`. JetBrains offers a 
CLI called `ijhttp` we can use.

We start by downloading it off the JetBrains pages:

```powershell
curl.exe -f -L -o ijhttp.zip "https://jb.gg/ijhttp/latest"
```

And extract it to a folder `Tools` in the User Profile:

```
Expand-Archive .\ijhttp.zip -DestinationPath "$env:USERPROFILE\Tools\ijhttp"
```

We can then add `ijhttp` to the search `Path` in Windows:

```powershell
$folder = "$env:USERPROFILE\Tools\ijhttp\ijhttp"
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")

[Environment]::SetEnvironmentVariable("Path", "$userPath;$folder", "User")
```

### The *.http File with the Requests ###


```java
@baseUrl = https://localhost:5000
@issueId = 12345
@delayMs = 30000

### Start the Agent Job
# @name startAgent
POST {{baseUrl}}/agent/start
Content-Type: application/json

{
  "issue_id": "{{issueId}}"
}

> {%
    client.test("Agent was started", function () {
        client.assert(response.status === 200, "Expected HTTP 200");
        client.assert(response.body.taskId, "Response does not contain taskId");
    });

    client.global.set("taskId", response.body.taskId);
    client.log("Stored taskId: " + response.body.taskId);
%}

### Reject the first attempt after a delay
< {%
    await sleep(Number(request.variables.get("delayMs")));
%}
POST {{baseUrl}}/agent/review/{{issueId}}/{{taskId}}-attempt-1
Content-Type: application/json

{
  "approved": false,
  "reason": "This is way too simple, add a better error handling strategy!"
}

> {%
    client.test("First review was submitted", function () {
        client.assert(response.status === 200, "Expected HTTP 200");
    });
%}

### Approve the second attempt after another delay
< {%
    await sleep(Number(request.variables.get("delayMs")));
%}
POST {{baseUrl}}/agent/review/{{issueId}}/{{taskId}}-attempt-2
Content-Type: application/json

{
  "approved": true,
  "reason": "Now, this looks good!"
}

> {%
    client.test("Second review was submitted", function () {
        client.assert(response.status === 200, "Expected HTTP 200");
    });
%}
```

### Analyzing the Log Output ###

After starting the Backend we can see the Postgres container being booted:

```
2026-08-16T10:37:08.743+02:00  INFO 24396 --- [           main] tc.postgres:18                           : Creating container for image: postgres:18
2026-08-16T10:37:08.807+02:00  INFO 24396 --- [           main] tc.postgres:18                           : Container postgres:18 is starting: b98a7b3788ed1e6cf7400ea0cc0d446c03f9aa36b61de449f4fd5e9dd7e8e82d
2026-08-16T10:37:09.877+02:00  INFO 24396 --- [           main] tc.postgres:18                           : Container postgres:18 started in PT1.1334854S
2026-08-16T10:37:09.878+02:00  INFO 24396 --- [           main] tc.postgres:18                           : Container is started (JDBC URL: jdbc:postgresql://localhost:44757/test?loggerLevel=OFF)
2026-08-16T10:37:09.884+02:00  INFO 24396 --- [           main] org.testcontainers.ext.ScriptUtils       : Executing database script from ssf-postgres.sql
2026-08-16T10:37:10.001+02:00  INFO 24396 --- [           main] org.testcontainers.ext.ScriptUtils       : Executed database script from ssf-postgres.sql in 116 ms.
```

And the Worker for the `ai-agent-queue` being created:

```
2026-08-16T10:37:10.141+02:00  INFO 24396 --- [    virtual-103] d.b.sqlflow.core.workers.SqlFlowWorker   : SqlFlow Worker [spring-worker-1] started for queue 'ai-agent-queue'
```

The Backend is ready to perform. So let's give it something to eat.

We'll switch to the IntelliJ Terminal:

* `File → Settings → Tools → Terminal`

We'll then run out `*.http` script using `ijhttp -L VERBOSE agent-requests.http`. 

The first request for fixing an issue `12345` is sent:

```
PS sqlflow-example\requests> ijhttp -L VERBOSE agent-requests.http
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Running IntelliJ HTTP Client with                      │
├────────────────────────┬────────────────────────────────────────────────────┤
│         Files          │ agent-requests.http                                │
├────────────────────────┼────────────────────────────────────────────────────┤
│   Public Environment   │                                                    │
├────────────────────────┼────────────────────────────────────────────────────┤
│  Private Environment   │                                                    │
└────────────────────────┴────────────────────────────────────────────────────┘
Request 'startAgent' POST http://localhost:5000/agent/start
= request =>
POST http://localhost:5000/agent/start
Content-Type: application/json
Content-Length: 25
User-Agent: IntelliJ HTTP Client/CLI 2026.1
Accept-Encoding: br, deflate, gzip, x-gzip
Accept: */*

{
  "issue_id": "12345"
}

###

<= response =
HTTP/1.1 200
Content-Type: application/json
Content-Length: 144
Date: Sun, 16 Aug 2026 08:44:33 GMT

{"taskId":"d51e000b-b1cc-4ac2-a30a-5521ae3f1c15","runId":"6b565cc1-872c-4cd1-b4b9-a84c5ba2e4b5","status":"Agent dispatched to fix Issue #12345"}

Response code: 200; Time: 448ms (448 ms); Content length: 144 bytes (144 B)
```

In the Backend we can see our fictional agent doing its fictional work:

```
2026-08-16T10:45:36.259+02:00  INFO 24396 --- [onPool-worker-3] d.b.s.e.workflows.AutonomousAgentJob     : Agent starts researching ticket 12345
2026-08-16T10:45:36.259+02:00  INFO 24396 --- [onPool-worker-3] d.b.s.e.workflows.AutonomousAgentJob     : Attempt 1/3: Generating a fix based on: Initial Attempt
2026-08-16T10:45:36.259+02:00  INFO 24396 --- [onPool-worker-3] d.b.s.e.workflows.AutonomousAgentJob     : Review for d51e000b-b1cc-4ac2-a30a-5521ae3f1c15-attempt-1 has been requested. Agent goes idle and waits for the code review...
```

We can see it goes idle and requests a human review. But the ficional fix looks way too simple, so we'll reject it:

```
Request 'Reject the first attempt after a delay' POST http://localhost:5000/agent/review/12345/d51e000b-b1cc-4ac2-a30a-5521ae3f1c15-attempt-1
= request =>
POST http://localhost:5000/agent/review/12345/d51e000b-b1cc-4ac2-a30a-5521ae3f1c15-attempt-1
Content-Type: application/json
Content-Length: 100
User-Agent: IntelliJ HTTP Client/CLI 2026.1
Accept-Encoding: br, deflate, gzip, x-gzip
Accept: */*

{
  "approved": false,
  "reason": "This is way too simple, add a better error handling strategy!"
}

###

<= response =
HTTP/1.1 200
Content-Type: application/json
Content-Length: 175
Date: Sun, 16 Aug 2026 08:45:05 GMT

{"message":"Fix for d51e000b-b1cc-4ac2-a30a-5521ae3f1c15-attempt-1 rejected. Agent tries again with feedback: 'This is way too simple, add a better error handling strategy!'"}

Response code: 200; Time: 26ms (26 ms); Content length: 175 bytes (175 B)
```

We can see the Backend receiving the request and the agent is generating another fix, based on our feedback:

```
2026-08-16T10:45:36.259+02:00  WARN 24396 --- [onPool-worker-3] d.b.s.e.workflows.AutonomousAgentJob     : Attempt 1 has been rejected: This is way too simple, add a better error handling strategy!
2026-08-16T10:45:36.259+02:00  INFO 24396 --- [onPool-worker-3] d.b.s.e.workflows.AutonomousAgentJob     : Attempt 2/3: Generating a fix based on: This is way too simple, add a better error handling strategy!
2026-08-16T10:45:36.259+02:00  INFO 24396 --- [onPool-worker-3] d.b.s.e.workflows.AutonomousAgentJob     : Review for d51e000b-b1cc-4ac2-a30a-5521ae3f1c15-attempt-2 has been requested. Agent goes idle and waits for the code review...
```

Let's not spend too many fictional tokens on this and accept the fix:

```
Request 'Approve the second attempt after another delay' POST http://localhost:5000/agent/review/12345/d51e000b-b1cc-4ac2-a30a-5521ae3f1c15-attempt-2
= request =>
POST http://localhost:5000/agent/review/12345/d51e000b-b1cc-4ac2-a30a-5521ae3f1c15-attempt-2
Content-Type: application/json
Content-Length: 59
User-Agent: IntelliJ HTTP Client/CLI 2026.1
Accept-Encoding: br, deflate, gzip, x-gzip
Accept: */*

{
  "approved": true,
  "reason": "Now, this looks good!"
}

###

<= response =
HTTP/1.1 200
Content-Type: application/json
Content-Length: 112
Date: Sun, 16 Aug 2026 08:45:35 GMT

{"message":"Fix for d51e000b-b1cc-4ac2-a30a-5521ae3f1c15-attempt-2 approved. Agent is now completing its work."}

Response code: 200; Time: 19ms (19 ms); Content length: 112 bytes (112 B)
```

In the logs we can see a happy agent completing the mission and creating a PR:

```
2026-08-16T10:45:36.259+02:00  INFO 24396 --- [onPool-worker-3] d.b.s.e.workflows.AutonomousAgentJob     : Fix approved. Creating Pull Request...
2026-08-16T10:45:36.261+02:00  INFO 24396 --- [onPool-worker-3] d.b.s.e.s.impl.DefaultGitHubService      : GitHub: PR for Issue #12345 has been created...
2026-08-16T10:45:37.467+02:00  INFO 24396 --- [onPool-worker-3] d.b.s.e.workflows.AutonomousAgentJob     : Mission accomplished, the PR has been created: https://github.com/company/repo/pull/7421
```
