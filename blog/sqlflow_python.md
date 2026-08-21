title: Introducing SqlFlow: Building Resilient Workflows with SqlFlow (Python SDK)
date: 2026-08-21 23:03
tags: python, sqlflow
category: python
slug: sqlflow_python
author: Philipp Wagner
summary: This article shows how to build an Autonomous AI Agent with SqlFlow

SqlFlow is a simple durable execution workflow system, for PostgreSQL and SQL Server. It handles scheduling 
and retries, without needing any other services to run in addition to PostgreSQL or SQL Server.

The SQL Script for creating the SqlFlow Database Schema is available here:

* [https://github.com/bytefish/SqlFlow/blob/main/sql/](https://github.com/bytefish/SqlFlow/blob/main/sql)

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

# Getting Started with the Python SDK #

Install the SqlFlow SDK from PyPI:

```bash
pip install sqlflow-sdk
```

If you want to use PostgreSQL support, install the PostgreSQL extra:

```bash
pip install "sqlflow-sdk[postgres]"
```

If you want to use SQL Server support, install the SQL Server extra:

```bash
pip install "sqlflow-sdk[sqlserver]"
```

To install all supported database drivers:

```bash
pip install "sqlflow-sdk[all]"
```

Then import the SDK in your application:

```python
from sqlflow import SqlFlow
from sqlflow.drivers.postgres import PostgresDriver
```

Or for SQL Server:

```python
from sqlflow import SqlFlow
from sqlflow.drivers.sqlserver import SqlServerDriver
```

Use either the PostgreSQL or SQL Server driver depending on your database backend.


# Python SDK: Building a Durable AI Agent #

## What we are going to build ##

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

## Building an Agent Job ##

To demonstrate how durable execution with SqlFlow works, we are going to build an autonomous AI agent that 
fixes bugs. The workflow is quickly laid out as: 

1. The agent receives a GitHub issue ID and fetches the stack trace.  
2. It generates a potential code fix using a Large Language Model (LLM).  
3. It pauses and asks a human for approval.  
4. If the human rejects the fix and provides feedback, the agent tries again (up to 3 times).  
5. If approved, it creates a Pull Request. If it fails 3 times, it escalates to a senior developer.

So first, let's define the data models that represent our inputs, states and final output:

```python
class AgentTask(BaseModel):
    issue_id: str

class Issue(BaseModel):
    stack_trace: str

class Solution(BaseModel):
    patched_code: str

class HumanApproval(BaseModel):
    approved: bool
    reason: Optional[str] = None

class AgentResult(BaseModel):
    success: bool
    pull_request_url: Optional[str] = None
    reason: Optional[str] = None
```

## The LLM Service ##

Next, we need a service to handle the AI code generation. In the real world, calling an LLM is a slow (and expensive) and the HTTP 
requests might fail or time out. We are wrapping these expensive calls with SqlFlow, so we don't lose all our state, if the 
server crashes.

For this demonstration, we are simulating ab LLM API call with some delay and return a hardcoded "code fixes" based on a reviewer's feedback:

```python
class LlmService:
    def __init__(self):
        self.logger = logging.getLogger("LlmService")

    async def generate_fix(self, log: str, last_feedback: str) -> dict:
        self.logger.info(f"Agent is thinking: 'Learned from feedback: {last_feedback}'")
        
        # Simulate a very expensive LLM call with a delay
        await asyncio.sleep(2.5)
        
        # Change Code based on human feedback
        if "error handling" in last_feedback.lower():
            code = "// AI: Improved Logging & Error-Handling added\nif(data == null) raise ValueError('Null data');"
        else:
            code = "// AI: Simple Fix for the NullReferenceException\nif(data is None): return"
            
        self.logger.info(f"LLM has generated a potential fix: {code}")
        
        # We return a dict to make it easily serializable by SqlFlow
        return Solution(patched_code=code).model_dump()
```

The agent needs to interact with the outside world. The GitHub service handles fetching the initial issue details and creating the final 
Pull Request. Whenever the LLM has generated has generated a solution, a human review is requested. If the LLM has been using more than 
a maximum amounts, the issue is escalated to a lead developer.

```python
class GitHubService:
    def __init__(self):
        self.logger = logging.getLogger("GitHubService")

    async def get_issue_details(self, issue_id: str) -> dict:
        self.logger.info(f"GitHub: Gets Ticket #{issue_id} details from the Repository...")
        await asyncio.sleep(0.8)
        return Issue(stack_trace="NullReferenceException at PaymentGateway.cs:42").model_dump()

    async def create_pull_request(self, issue_id: str, code: str) -> str:
        self.logger.info(f"GitHub: PR for Issue #{issue_id} has been created...")
        await asyncio.sleep(1.2)
        return f"https://github.com/company/repo/pull/{random.randint(1000, 9999)}"

    async def escalate_to_senior(self, issue_id: str, reason: str) -> None:
        self.logger.critical(f"ESCALATION to Senior Developer: Issue #{issue_id} - Reason: {reason}")
        await asyncio.sleep(0.5)

    async def request_human_review(self, issue_id: str, proposed_fix: dict, correlation_id: str) -> None:
        patched_code = proposed_fix.get("patched_code", "")
        self.logger.info(f"ACTION REQUIRED: Solution for Issue #{issue_id} with Correlation-ID {correlation_id} has been created: {patched_code}...")
        await asyncio.sleep(1.2)
```

## The Autonomous Agent Job ##

The workflow is just a normal Python method, that takes a `TaskContext` and parameters. The magic is in the `ctx.step` method: every time a step completes, 
its result is automatically checkpointed to the Postgres database. If the process crashes or is restarted, the framework replays the job. It skips the already 
completed steps and loads their results directly from the database.

And then instead of blocking a thread by sleeping or an infinite polling loop, we use `ctx.await_event` to wait for human interaction. This 
instructs the engine to safely suspend the workflow state to the database and free up the worker until an external system fires the specific event 
being awaited.

```python
async def autonomous_agent_workflow(ctx: TaskContext, params: dict) -> dict:
    logger = logging.getLogger("AutonomousAgentJob")
    task = AgentTask(**params)
    
    logger.info(f"Agent starts researching ticket {task.issue_id}")

    # Helper async functions for steps (since lambda doesn't work well with await in ctx.step)
    async def fetch_issue():
        return await github_service.get_issue_details(task.issue_id)
        
    # Load the Issue Context first, so the LLM has all relevant information
    bug_report_dict = await ctx.step("fetch-issue-context", fetch_issue)
    bug_report = Issue(**bug_report_dict)

    is_approved = False
    attempt = 0
    last_feedback = "Initial Attempt"

    while not is_approved and attempt < 3:
        attempt += 1
        correlation_id = f"{ctx.task_id}-attempt-{attempt}"
        
        logger.info(f"Attempt {attempt}/3: Generating a fix based on: {last_feedback}")

        async def generate_code():
            return await llm_service.generate_fix(bug_report.stack_trace, last_feedback)
            
        proposed_fix_dict = await ctx.step(f"generate-code-fix-{attempt}", generate_code)
        proposed_fix = Solution(**proposed_fix_dict)

        async def notify():
            await github_service.request_human_review(task.issue_id, proposed_fix.model_dump(), correlation_id)
            await notification_service.notify_reviewer(task.issue_id, correlation_id)
            return True # Step needs to return something serializable
            
        await ctx.step(f"notify-reviewer-{attempt}", notify)

        logger.info(f"Review for {correlation_id} has been requested. Agent goes idle and waits for the code review...")

        # Wait for a human decision without blocking a thread
        # This will throw SuspendTaskException if the event 
        # hasn't happened yet!
        review_data = await ctx.await_event(
            event_name=f"agent-approval:{task.issue_id}:{correlation_id}",
            step_name=f"wait-for-human-review-{attempt}"
        )

        approval = HumanApproval(**review_data)
        is_approved = approval.approved
        last_feedback = approval.reason or "No feedback has been given"

        if not is_approved:
            logger.warning(f"Attempt {attempt} has been rejected: {last_feedback}")

    if is_approved:
        logger.info("Fix approved. Creating Pull Request...")

        async def create_pr():
            return await github_service.create_pull_request(task.issue_id, "apply-fix")
            
        pr_url = await ctx.step("create-pull-request", create_pr)
        logger.info(f"Mission accomplished, the PR has been created: {pr_url}")
        
        return AgentResult(success=True, pull_request_url=pr_url).model_dump()
    else:
        logger.error(f"Maximum number of attempts reached. Escalates ticket {task.issue_id} to a human.")

        async def escalate():
            await github_service.escalate_to_senior(task.issue_id, "Agent didn't find a solution after 3 attempts.")
            return True
            
        await ctx.step("notify-senior-developer", escalate)
        
        return AgentResult(success=False, reason="Escalated to human supervisor after 3 failures.").model_dump()
```

A FastAPI application then serves as the host for our SqlFlow runtime.

During application startup, we:

* Create a PostgreSQL driver and establish the database connection.
* Create a SqlFlow client instance.
* Create or verify the workflow queue.
* Register the workflow implementation under a task name.
* Start a worker that continuously polls the queue and executes tasks.

The worker runs in the background alongside the FastAPI application. It continuously claims available tasks 
from the queue, executes workflow steps, persists checkpoints, and resumes suspended workflows when events 
arrive.

```python
# Web API (FastAPI) and Application Setup

logging.basicConfig(level=logging.INFO)

# Global variables to hold our DB driver, Client and Worker

db_driver = None
sqlflow_client = None
worker = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Lifecycle manager for the FastAPI application."""
    global db_driver, sqlflow_client, worker
    
    connection_string = "postgresql://postgres:password@127.0.0.1:5432/sqlflow_db"
    
    db_driver = PostgresDriver(connection_string)

    await db_driver.connect()
    
    sqlflow_client = SqlFlow(db=db_driver)
    
    await sqlflow_client.create_queue("ai-agent-queue")
    
    # Register Workflow
    sqlflow_client.register_task("solve-bug", autonomous_agent_workflow, max_attempts=3)
    
    # Start Worker
    worker = sqlflow_client.create_worker(WorkerOptions(
        worker_id="agent-worker-1",
        queue_name="ai-agent-queue",
        poll_interval=1.0,
        concurrency=1
    ))
    
    await worker.start()
    
    yield # App runs here
    
    # Cleanup on shutdown
    if worker:
        await worker.stop()
    if db_driver:
        await db_driver.disconnect()

app = FastAPI(lifespan=lifespan)
```

And to interact with the application, we'll add a set of HTTP endpoints for starting and interacting 
with the system. Again it uses the `sqlflow_client` abstraction to simplify working with SqlFlow. 

```python
@app.post("/agent/start")
async def start_agent(task: AgentTask):
    """A Webhook triggers the Agent, such as a new JIRA ticket or GitHub issue."""
    options = SpawnOptions(queue_name="ai-agent-queue")
    
    result = await sqlflow_client.spawn(
        options=options, 
        task_name="solve-bug", 
        params=task.model_dump()
    )
    
    return {
        "run_id": result.run_id, 
        "task_id": result.task_id,
        "status": f"Agent dispatched to fix Issue #{task.issue_id}"
    }

@app.post("/agent/review/{issue_id}/{correlation_id}")
async def review_agent(issue_id: str, correlation_id: str, approval: HumanApproval):
    """A Lead-Developer clicks on 'Approve' or 'Reject', with Feedback."""
    
    # Wake up the agent that is working on the ticket
    event_name = f"agent-approval:{issue_id}:{correlation_id}"
    
    options = EmitEventOptions(queue_name="ai-agent-queue")
    await sqlflow_client.emit_event(
        options=options, 
        event_name=event_name, 
        payload=approval.model_dump()
    )
    
    message = (
        f"Fix for {correlation_id} approved. Agent is now completing its work."
        if approval.approved else
        f"Fix for {correlation_id} rejected. Agent tries again with feedback: '{approval.reason}'"
    )
    
    return {"message": message}
```

## An Example Session with the AI Agent Job ##

### Getting the Tooling right ###

It's not stone age. I want to use tooling to fire my HTTP requests. There's somewhat of a 
standard established for firing HTTP Requests, which is the `*.http` format. 

I am not an expert with Python, so I'll use a CLI provided by JetBrains called `ijhttp`, 
that makes it super easy to work with HTTP Requests.

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
@baseUrl = http://localhost:8000
@issueId = 12345
@delayMs = 15000

### Start the Agent Job
# @name startAgent
POST {{baseUrl}}/agent/start
Content-Type: application/json

{
  "issue_id": "{{issueId}}"
}

> {%
    let body = response.body;
    if (typeof body === "string") {
        body = JSON.parse(body);
    }

    client.test("Agent was started", function () {
        client.assert(response.status === 200, "Expected HTTP 200");
        client.assert(body.task_id, "Response does not contain task_id");
    });

    client.global.set("task_id", body.task_id);
    client.log("Stored task_id: " + body.task_id);
%}

### Reject the first attempt after a delay
< {%
    // Directly use the evaluated template variable or fallback to 15000
    await sleep(parseInt("{{delayMs}}") || 15000);
%}
POST {{baseUrl}}/agent/review/{{issueId}}/{{task_id}}-attempt-1
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
    await sleep(parseInt("{{delayMs}}") || 15000);
%}
POST {{baseUrl}}/agent/review/{{issueId}}/{{task_id}}-attempt-2
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

### Starting the Log Output ###

We'll start the Backend by running:

```bash
poetry run uvicorn examples.ai_agent_api:app --reload
```

After starting the Backend we can see the Postgres container being booted, a Postgres connection check and the worker queue 
being created:

```
INFO:     Will watch for changes in these directories: ['C:\\Users\\philipp\\source\\repos\\bytefish\\SqlFlowCore\\sdks\\python']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [24964] using StatReload
INFO:     Started server process [13256]
INFO:     Waiting for application startup.
INFO:sqlflow.postgres:PostgreSQL connection pool created.
INFO:sqlflow:Worker agent-worker-1 started on queue 'ai-agent-queue'.
INFO:     Application startup complete.
```

The Backend is ready to perform. So let's give it something to eat.

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
date: Fri, 21 Aug 2026 20:56:51 GMT
server: uvicorn
content-length: 146
content-type: application/json

{"run_id":"15fa7c33-93c6-4179-92ed-2c7bd7001627","task_id":"010f6fee-2fa7-42e1-8c91-389ce54c68f2","status":"Agent dispatched to fix Issue #12345"}

Response code: 200 (OK); Time: 454ms (454 ms); Content length: 146 bytes (146 B)

```

In the Backend we can see our fictional agent doing its fictional work:

```
INFO:     127.0.0.1:57179 - "POST /agent/start HTTP/1.1" 200 OK
INFO:AutonomousAgentJob:Agent starts researching ticket 12345
INFO:GitHubService:GitHub: Gets Ticket #12345 details from the Repository...
INFO:AutonomousAgentJob:Attempt 1/3: Generating a fix based on: Initial Attempt
INFO:LlmService:Agent is thinking: 'Learned from feedback: Initial Attempt'
INFO:LlmService:LLM has generated a potential fix: // AI: Simple Fix for the NullReferenceException
if(data is None): return
INFO:GitHubService:ACTION REQUIRED: Solution for Issue #12345 with Correlation-ID 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-1 has been created: // AI: Simple Fix for the NullReferenceException
if(data is None): return...
INFO:LocalNotification:Ping! Please review 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-1 for issue 12345.
INFO:AutonomousAgentJob:Review for 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-1 has been requested. Agent goes idle and waits for the code review...
INFO:sqlflow:Task 010f6fee-2fa7-42e1-8c91-389ce54c68f2 suspended: Task suspended waiting for event: agent-approval:12345:010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-1
```

We can see it goes idle and requests a human review. But the ficional fix looks way too simple, so we'll reject it:

```

Request 'Reject the first attempt after a delay' POST http://localhost:8000/agent/review/12345/010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-1
= request =>
POST http://localhost:8000/agent/review/12345/010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-1
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
date: Fri, 21 Aug 2026 20:57:22 GMT
server: uvicorn
content-length: 175
content-type: application/json

{"message":"Fix for 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-1 rejected. Agent tries again with feedback: 'This is way too simple, add a better error handling strategy!'"}

Response code: 200 (OK); Time: 19ms (19 ms); Content length: 175 bytes (175 B)
```

We can see the Backend receiving the request and the agent is generating another fix, based on our feedback:

```
INFO:     127.0.0.1:57182 - "POST /agent/review/12345/010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-1 HTTP/1.1" 200 OK
INFO:AutonomousAgentJob:Agent starts researching ticket 12345
INFO:AutonomousAgentJob:Attempt 1/3: Generating a fix based on: Initial Attempt
INFO:AutonomousAgentJob:Review for 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-1 has been requested. Agent goes idle and waits for the code review...
WARNING:AutonomousAgentJob:Attempt 1 has been rejected: This is way too simple, add a better error handling strategy!
INFO:AutonomousAgentJob:Attempt 2/3: Generating a fix based on: This is way too simple, add a better error handling strategy!
INFO:LlmService:Agent is thinking: 'Learned from feedback: This is way too simple, add a better error handling strategy!'
INFO:LlmService:LLM has generated a potential fix: // AI: Improved Logging & Error-Handling added
if(data == null) raise ValueError('Null data');
INFO:GitHubService:ACTION REQUIRED: Solution for Issue #12345 with Correlation-ID 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-2 has been created: // AI: Improved Logging & Error-Handling added
if(data == null) raise ValueError('Null data');...
INFO:LocalNotification:Ping! Please review 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-2 for issue 12345.
INFO:AutonomousAgentJob:Review for 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-2 has been requested. Agent goes idle and waits for the code review...
INFO:sqlflow:Task 010f6fee-2fa7-42e1-8c91-389ce54c68f2 suspended: Task suspended waiting for event: agent-approval:12345:010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-2
```

Let's not spend too many fictional tokens on this and accept the fix:

```

Request 'Approve the second attempt after another delay' POST http://localhost:8000/agent/review/12345/010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-2
= request =>
POST http://localhost:8000/agent/review/12345/010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-2
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
date: Fri, 21 Aug 2026 20:57:53 GMT
server: uvicorn
content-length: 112
content-type: application/json

{"message":"Fix for 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-2 approved. Agent is now completing its work."}

Response code: 200 (OK); Time: 21ms (21 ms); Content length: 112 bytes (112 B)
```

In the logs we can see a happy agent completing the mission and creating a PR:

```
WARNING:AutonomousAgentJob:Attempt 1 has been rejected: This is way too simple, add a better error handling strategy!
INFO:AutonomousAgentJob:Attempt 2/3: Generating a fix based on: This is way too simple, add a better error handling strategy!
INFO:AutonomousAgentJob:Review for 010f6fee-2fa7-42e1-8c91-389ce54c68f2-attempt-2 has been requested. Agent goes idle and waits for the code review...
INFO:AutonomousAgentJob:Fix approved. Creating Pull Request...
INFO:GitHubService:GitHub: PR for Issue #12345 has been created...
INFO:AutonomousAgentJob:Mission accomplished, the PR has been created: https://github.com/company/repo/pull/7272
INFO:sqlflow:Task 010f6fee-2fa7-42e1-8c91-389ce54c68f2 completed successfully.```
