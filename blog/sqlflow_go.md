title: Introducing SqlFlow: Building Resilient Workflows with SqlFlow (Go SDK)
date: 2026-08-23 23:10
tags: go, sqlflow
category: go
slug: sqlflow_go
author: Philipp Wagner
summary: This article shows how to build an Autonomous AI Agent with SqlFlow

SqlFlow is a simple durable execution workflow system, for PostgreSQL and SQL Server. It handles scheduling 
and retries, without needing any other services to run in addition to PostgreSQL or SQL Server.

The SQL Script for creating the SqlFlow Database Schema is available here:

* [https://github.com/bytefish/SqlFlow/blob/main/sql/](https://github.com/bytefish/SqlFlow/blob/main/sql)

It took a large deal of inspiration from [Absurd], but features a simpler database model. 

[Absurd]: https://github.com/earendil-works/absurd

# Table of contents #

[TOC]

# Getting Started with the Database #

You'll start by creating the `ssf` database schema and tables:

* PostgreSQL: `sql/ssf-postgres.sql`
* SQL Server: `sql/ssf-sqlserver.sql`

# Getting Started with the Go SDK #

Install the SqlFlow Go SDK using go get:

```bash
go get github.com/bytefish/SqlFlow/sdks/go
```

Then import the SDK and your preferred database driver in your application.

For PostgreSQL support, also import the postgres subpackage:

```go
import (
	"github.com/bytefish/SqlFlow/sdks/go"
	"github.com/bytefish/SqlFlow/sdks/go/postgres"
)
```

# Go SDK: Building a Durable AI Agent #

## What we are going to build ##

The classic examples for durable execution are usually e-commerce checkouts or payment processing scenarios. But there's another rapidly 
growing use case developers are dealing with: Autonomous AI Agents. Building AI agents that interact with external APIs, write code, 
or execute complex workflows introduces challenges.

1. LLM API calls are inherently slow, prone to timeouts or rate limits. And they are also quite expensive, right? If a server crashes 
or restarts while waiting for a 30-second AI generation, standard async and await state is lost forever. 
2. You don't want an AI to push code to production or execute financial transactions without a human looking at it. Agents need to pause 
their execution, ask a human for permission and resume only when approved. This is sometimes hours or days later.

Traditional approaches require you to build complex state machines, database polling loops, or heavy external infrastructure. With SqlFlow, 
we can write our agent as standard, sequential Go code. The framework will automatically checkpoint the state to Postgres, sleep without blocking 
server threads, and wake up exactly where it left off.

## Building an Agent Job ##

To demonstrate how durable execution with SqlFlow works, we are going to build an autonomous AI agent that 
fixes bugs. The workflow is quickly laid out as: 

1. The agent receives a GitHub issue ID and fetches the stack trace.  
2. It generates a potential code fix using a Large Language Model (LLM).  
3. It pauses and asks a human for approval.  
4. If the human rejects the fix and provides feedback, the agent tries again (up to 3 times).  
5. If approved, it creates a Pull Request. If it fails 3 times, it escalates to a senior developer.

So first, let's define the data models that represent our inputs, states and final output:

```go
type AgentTask struct {
	IssueID string `json:"issue_id"`
}

type Issue struct {
	StackTrace string `json:"stack_trace"`
}

type Solution struct {
	PatchedCode string `json:"patched_code"`
}

type HumanApproval struct {
	Approved bool   `json:"approved"`
	Reason   string `json:"reason,omitempty"`
}

type AgentResult struct {
	Success        bool   `json:"success"`
	PullRequestURL string `json:"pull_request_url,omitempty"`
	Reason         string `json:"reason,omitempty"`
}
```

## The LLM Service ##

Next, we need a service to handle the AI code generation. In the real world, calling an LLM is a slow (and expensive) and the HTTP 
requests might fail or time out. We are wrapping these expensive calls with SqlFlow, so we don't lose all our state, if the 
server crashes.

For this demonstration, we are simulating an LLM API call with `time.Sleep` and return hardcoded "code fixes" based on a reviewer's 
feedback. Because of Go's goroutines, `time.Sleep` only blocks the current worker thread.

```go

type LlmService struct{}

func (s *LlmService) GenerateFix(stackTrace string, lastFeedback string) Solution {
	log.Printf("[LlmService] Agent is thinking: 'Learned from feedback: %s'", lastFeedback)
	time.Sleep(2500 * time.Millisecond)

	var code string
	if strings.Contains(strings.ToLower(lastFeedback), "error handling") {
		code = "// AI: Improved Logging & Error Handling added\nif(data == null) raise ValueError('Null data');"
	} else {
		code = "// AI: Simple Fix for the NullReferenceException\nif(data is None): return"
	}

	log.Printf("[LlmService] LLM generated a potential fix:\n%s", code)
    
	return Solution{PatchedCode: code}
}
```

The agent needs to interact with the outside world. The GitHub service handles fetching the initial issue details and creating the final 
Pull Request. Whenever the LLM has generated has generated a solution, a human review is requested. If the LLM has been using more than 
a maximum amounts, the issue is escalated to a lead developer.

```go
type GitHubService struct{}

func (s *GitHubService) GetIssueDetails(issueID string) Issue {
	log.Printf("[GitHubService] Fetching details for ticket #%s from the repository...", issueID)
	time.Sleep(800 * time.Millisecond)
	return Issue{StackTrace: "NullReferenceException at PaymentGateway.cs:42"}
}

func (s *GitHubService) CreatePullRequest(issueID string, code string) string {
	log.Printf("[GitHubService] PR for issue #%s has been created...", issueID)
	time.Sleep(1200 * time.Millisecond)
	return fmt.Sprintf("https://github.com/company/repo/pull/%d", rand.Intn(9000)+1000)
}

func (s *GitHubService) EscalateToSenior(issueID string, reason string) {
	log.Printf("[GitHubService] 🚨 ESCALATION to Senior Developer: Issue #%s - Reason: %s", issueID, reason)
	time.Sleep(500 * time.Millisecond)
}

func (s *GitHubService) RequestHumanReview(issueID string, proposedFix Solution, correlationID string) {
	log.Printf("[GitHubService] ⏳ ACTION REQUIRED: A solution for issue #%s (Correlation ID %s) is available:\n%s",
		issueID, correlationID, proposedFix.PatchedCode)
	time.Sleep(1200 * time.Millisecond)
}
```

## The Autonomous Agent Job ##

The workflow is just a normal Go function that takes a `sqlflow.TaskContext` and your parameters. The magic is in the `sqlflow.Step`function: 
every time a step completes, its result is automatically checkpointed to the Postgres database. If the process crashes or is restarted, the 
framework replays the job. It skips the already completed steps and loads their results directly from the database.

Instead of blocking a goroutine with an infinite polling loop, we use `sqlflow.AwaitEvent[T]` to wait for human interaction. This instructs the 
engine to safely suspend the workflow state to the database and free up the worker completely until an external system fires the specific 
event being awaited.

```go

var llmService = &LlmService{}
var gitHubService = &GitHubService{}
var notificationService = &LocalNotificationService{}

func autonomousAgentWorkflow(ctx *sqlflow.TaskContext, task AgentTask) error {
	log.Printf("[Workflow] Agent starts investigation for ticket %s", task.IssueID)

	bugReport, err := sqlflow.Step(ctx, "fetch-issue-context", func() (Issue, error) {
		return gitHubService.GetIssueDetails(task.IssueID), nil
	})
	if err != nil {
		return err
	}

	isApproved := false
	attempt := 0
	lastFeedback := "Initial Attempt"

	for !isApproved && attempt < 3 {
		attempt++
		correlationID := fmt.Sprintf("%s-attempt-%d", ctx.TaskID, attempt)
		log.Printf("[Workflow] Attempt %d/3: Generating fix based on: '%s'", attempt, lastFeedback)

		proposedFix, err := sqlflow.Step(ctx, fmt.Sprintf("generate-code-fix-%d", attempt), func() (Solution, error) {
			return llmService.GenerateFix(bugReport.StackTrace, lastFeedback), nil
		})
		if err != nil {
			return err
		}

		_, err = sqlflow.Step(ctx, fmt.Sprintf("notify-reviewer-%d", attempt), func() (bool, error) {
			gitHubService.RequestHumanReview(task.IssueID, proposedFix, correlationID)
			notificationService.NotifyReviewer(task.IssueID, correlationID)
			return true, nil
		})

		if err != nil {
			return err
		}

		log.Printf("[Workflow] Review requested for %s. Agent goes to sleep and waits...", correlationID)

		approval, err := sqlflow.AwaitEvent[HumanApproval](
			ctx,
			fmt.Sprintf("agent-approval:%s:%s", task.IssueID, correlationID),
			fmt.Sprintf("wait-for-human-review-%d", attempt),
			nil,
		)
		if err != nil {
			return err
		}

		isApproved = approval.Approved
		if approval.Reason != "" {
			lastFeedback = approval.Reason
		} else {
			lastFeedback = "No feedback has been given"
		}

		if !isApproved {
			log.Printf("[Workflow] ❌ Attempt %d was rejected: %s", attempt, lastFeedback)
		}
	}

	if isApproved {
		log.Println("[Workflow] ✅ Fix approved! Creating pull request...")

		prURL, err := sqlflow.Step(ctx, "create-pull-request", func() (string, error) {
			return gitHubService.CreatePullRequest(task.IssueID, "apply-fix"), nil
		})
		if err != nil {
			return err
		}

		log.Printf("[Workflow] Mission accomplished! PR created: %s", prURL)
		return nil
	}

	log.Printf("[Workflow] 🚨 Maximum attempts reached. Escalating ticket %s to a human.", task.IssueID)
	_, err = sqlflow.Step(ctx, "notify-senior-developer", func() (bool, error) {
		gitHubService.EscalateToSenior(task.IssueID, "Agent could not find a solution after 3 attempts.")
		return true, nil
	})

	return err
}
```

## Interacting with the System: Providing HTTP Endpoints ##

Now there are two HTTP endpoints for interacting with the Workflow: 

* The `/agent/start` endpoint kicks off the process asynchronously and returns immediately.
* The `/agent/review/...` endpoint acts as our callback webhook. 
    * When the human reviewer approves or rejects a fix, this endpoint emits the event back into the queue, which then wakes up the sleeping job right where it left off.

The standard Go `net/http` application serves as the host for our SqlFlow runtime.

During application startup, we:

* Create a PostgreSQL driver and establish the database connection.
* Create a SqlFlow client instance.
* Create or verify the workflow queue.
* Register the workflow implementation under a task name.
Ü Start a worker that continuously polls the queue and executes tasks.

The worker runs in the background using goroutines alongside the HTTP server. It continuously claims available tasks from the 
queue, executes workflow steps, persists checkpoints, and resumes suspended workflows when events arrive.

```go
var sqlflowClient *sqlflow.Client

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()

	connString := "postgres://postgres:password@localhost:5432/sqlflow_db"
	dbDriver, err := postgres.NewPostgresDriver(ctx, connString)
	if err != nil {
		log.Fatalf("Error connecting to the database: %v", err)
	}
	defer dbDriver.Close()

	sqlflowClient = sqlflow.NewClient(dbDriver)
	_ = sqlflowClient.CreateQueue(ctx, "ai-agent-queue", "unpartitioned")

	sqlflow.RegisterWorkflow(sqlflowClient, "solve-bug", autonomousAgentWorkflow)

	workerOpts := sqlflow.WorkerOptions{
		WorkerID:     "agent-worker-1",
		QueueName:    "ai-agent-queue",
		PollInterval: 1 * time.Second,
		Concurrency:  5,
	}
	
	worker := sqlflowClient.CreateWorker(workerOpts)
	worker.Start(ctx)
	
	log.Println("Background worker started.")

	mux := http.NewServeMux()

	mux.HandleFunc("POST /agent/start", func(w http.ResponseWriter, r *http.Request) {
		var task AgentTask
		if err := json.NewDecoder(r.Body).Decode(&task); err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}

		options := sqlflow.SpawnOptions{QueueName: "ai-agent-queue"}
		res, err := sqlflowClient.Spawn(r.Context(), options, "solve-bug", task)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]any{
			"run_id":  res.RunID,
			"task_id": res.TaskID,
			"status":  fmt.Sprintf("Agent dispatched to fix Issue #%s", task.IssueID),
		})
	})

	mux.HandleFunc("POST /agent/review/{issue_id}/{correlation_id}", func(w http.ResponseWriter, r *http.Request) {
		issueID := r.PathValue("issue_id")
		correlationID := r.PathValue("correlation_id")

		var approval HumanApproval
		if err := json.NewDecoder(r.Body).Decode(&approval); err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}

		eventName := fmt.Sprintf("agent-approval:%s:%s", issueID, correlationID)
		options := sqlflow.EmitEventOptions{QueueName: "ai-agent-queue"}
		
		err := sqlflowClient.EmitEvent(r.Context(), options, eventName, approval)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}

		msg := fmt.Sprintf("Fix for %s rejected. Agent will try again using feedback: '%s'", correlationID, approval.Reason)
		if approval.Approved {
			msg = fmt.Sprintf("Fix for %s approved. Agent is completing the work.", correlationID)
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"message": msg})
	})

	server := &http.Server{
		Addr:    ":8000",
		Handler: mux,
	}

	go func() {
		log.Println("Web API is running at http://localhost:8000")
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("HTTP Server Fehler: %v", err)
		}
	}()

	<-ctx.Done()
	log.Println("Shutting down system...")

	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer shutdownCancel()
	
	server.Shutdown(shutdownCtx)
	worker.Stop()
	time.Sleep(500 * time.Millisecond)
	
	log.Println("System shut down successfully.")
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

### Analyzing the Session ###

We start by running our Go project:

```bash
go run main.go
```

We'll then run our *.http script using ijhttp -L VERBOSE agent-requests.http.

The first request for fixing an issue 12345 is sent:

```bash
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
Request 'startAgent' POST http://localhost:8000/agent/start
= request =>
POST http://localhost:8000/agent/start
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
HTTP/1.1 200 OK
date: Sun, 23 Aug 2026 09:25:51 GMT
server: Go-http-client/1.1
content-length: 146
content-type: application/json

{"run_id":"15fa7c33-93c6-4179-92ed-2c7bd7001627","task_id":"27a11757-3ff2-4baa-9092-5e924dba5a6f","status":"Agent dispatched to fix Issue #12345"}

Response code: 200 (OK); Time: 454ms (454 ms); Content length: 146 bytes (146 B)
```

In the Go backend output, we can see our fictional agent doing its work. It goes idle and requests a human review. We'll reject the fix.

```bash
Request 'Reject the first attempt after a delay' POST http://localhost:8000/agent/review/12345/27a11757-3ff2-4baa-9092-5e924dba5a6f-attempt-1
= request =>
POST http://localhost:8000/agent/review/12345/27a11757-3ff2-4baa-9092-5e924dba5a6f-attempt-1
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
HTTP/1.1 200 OK
date: Sun, 23 Aug 2026 09:26:22 GMT
server: Go-http-client/1.1
content-length: 175
content-type: application/json

{"message":"Fix for 27a11757-3ff2-4baa-9092-5e924dba5a6f-attempt-1 rejected. Agent tries again with feedback: 'This is way too simple, add a better error handling strategy!'"}

Response code: 200 (OK); Time: 19ms (19 ms); Content length: 175 bytes (175 B)
```

We can see the Go worker waking up, restoring state and generating another fix based on our feedback. Let's accept the fix:

```bash
Request 'Approve the second attempt after another delay' POST http://localhost:8000/agent/review/12345/27a11757-3ff2-4baa-9092-5e924dba5a6f-attempt-2
= request =>
POST http://localhost:8000/agent/review/12345/27a11757-3ff2-4baa-9092-5e924dba5a6f-attempt-2
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
HTTP/1.1 200 OK
date: Sun, 23 Aug 2026 09:26:53 GMT
server: Go-http-client/1.1
content-length: 112
content-type: application/json

{"message":"Fix for 27a11757-3ff2-4baa-9092-5e924dba5a6f-attempt-2 approved. Agent is now completing its work."}

Response code: 200 (OK); Time: 21ms (21 ms); Content length: 112 bytes (112 B)
```

In the Go backend logs, we can observe the entire lifecycle: how the workflow is suspended, fully unloaded from 
memory, and then replayed from the database checkpoints exactly when our http requests arrive:

```bash
2026/08/23 09:25:50 Worker agent-worker-1 started on queue 'ai-agent-queue'
2026/08/23 09:25:50 Background worker started.
2026/08/23 09:25:50 Web API is running at http://localhost:8000
2026/08/23 09:27:02 [Workflow] Agent starts investigation for ticket 12345
2026/08/23 09:27:02 [Workflow] Attempt 1/3: Generating fix based on: 'Initial Attempt'
2026/08/23 09:27:02 [Workflow] Review requested for 27a11757-3ff2-4baa-9092-5e924dba5a6f-attempt-1. Agent goes to sleep and waits...
2026/08/23 09:27:02 [Workflow] ❌ Attempt 1 was rejected: This is way too simple, add a better error handling strategy!
2026/08/23 09:27:02 [Workflow] Attempt 2/3: Generating fix based on: 'This is way too simple, add a better error handling strategy!'
2026/08/23 09:27:02 [Workflow] Review requested for 27a11757-3ff2-4baa-9092-5e924dba5a6f-attempt-2. Agent goes to sleep and waits...
2026/08/23 09:27:02 [Workflow] ✅ Fix approved! Creating pull request...
2026/08/23 09:27:02 [GitHubService] PR for issue #12345 has been created...
2026/08/23 09:27:03 [Workflow] Mission accomplished! PR created: https://github.com/company/repo/pull/7971
```