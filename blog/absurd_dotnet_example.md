title: Durable AI Agents: Building Resilient Workflows with Absurd.NET
date: 2026-07-04 17:12
tags: dotnet
category: dotnet
slug: absurd_dotnet_example
author: Philipp Wagner
summary: This article shows how to build an Autonomous AI Agent with Absurd.NET

In this article I want to introduce you to Absurd.NET, which is a .NET SDK built for the eponymous Durable Execution Workflow System:

> Absurd is the simplest durable execution workflow system you can think of. It's entirely based 
> on Postgres and nothing else. It's almost as easy to use as a queue, but it handles scheduling 
> and retries, and it does all of that without needing any other services to run in addition 
> to Postgres.

All SQL scripts for creating the Absurd workflow system inside your Postgres database are available at:

* [https://github.com/earendil-works/absurd](https://github.com/earendil-works/absurd)

All code is available in a Git repository at:

* [https://github.com/bytefish/Absurd.NET](https://github.com/bytefish/Absurd.NET)

## Table of contents ##

[TOC]

## What we are going to build ##

The classic examples for durable execution are usually e-commerce checkouts or payment processing scenarios. But there's an interesting use case developers are dealing with: Autonomous AI Agents. 

Building AI agents that interact with external APIs, write code or execute complex workflows introduces challenges.

1. LLM APIs are slow and prone to timeouts or rate limits. They are also quite expensive, right? So if a server crashes, you want to keep the state.
2. You don't want AI to push code to production without human oversight. This is sometimes hours or days later.

Traditional approaches require you to build complex state machines, database polling loops and lots of infrastructure code. 

With Absurd.NET we can write our agent as standard, sequential C# code. The framework will automatically checkpoint the state to Postgres, sleep without blocking server threads and wake up exactly where it left off.

## Building an Agent Job ##

To demonstrate how durable execution with Absurd.NET works, we are going to build an autonomous AI agent that 
fixes bugs. 

The workflow is laid out as: 

1. The agent receives a GitHub issue ID and fetches the stack trace.  
2. It generates a potential code fix using a Large Language Model (LLM).  
3. It pauses and asks a human for approval.  
4. If the human rejects the fix and provides feedback, the agent tries again (up to 3 times).  
5. If approved, it creates a Pull Request. If it fails 3 times, it escalates to a senior developer.

So first, let's define the data models that represent our inputs, states and final output:

```csharp
public class AgentTask 
{
    [JsonPropertyName("issue_id")]
    public string IssueId { get; set; } = ""; 
}

public class Issue 
{
    [JsonPropertyName("stack_trace")]
    public string StackTrace { get; set; } = ""; 
}

public class Solution 
{
    [JsonPropertyName("patched_code")]
    public string PatchedCode { get; set; } = ""; 
}

public class HumanApproval 
{
    [JsonPropertyName("approved")]
    public bool Approved { get; set; }

    [JsonPropertyName("reason")]
    public string? Reason { get; set; } 
}

public class AgentResult
{
    [JsonPropertyName("success")]
    public bool Success { get; set; }

    [JsonPropertyName("pull_request_url")]
    public string? PullRequestUrl { get; set; }

    [JsonPropertyName("reason")]
    public string? Reason { get; set; }
}
```

## The LLM Service ##

We need a service to handle the AI code generation. 

We are wrapping the expensive calls to the LLM with Absurd.NET, so we don't lose all our state, if the server crashes.

For this demonstration, we are simulating ab LLM API call with some delay and return a hardcoded "code fixes" based on a reviewer's feedback:

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

using AbsurdSdk.AiSample.Models;

namespace AbsurdSdk.AiSample.Services;

public interface ILlmService
{
    Task<Solution> GenerateFixAsync(string log, string lastFeedback, CancellationToken ct);
}

public class LlmService : ILlmService
{
    private readonly ILogger<LlmService> _logger;
    public LlmService(ILogger<LlmService> logger) => _logger = logger;

    public async Task<Solution> GenerateFixAsync(string log, string lastFeedback, CancellationToken ct)
    {
        _logger.LogInformation("Agent is thinking: 'Learned from feedback: {feedback}'", lastFeedback);

        // Simulate a very expensive LLM call with a delay
        await Task.Delay(2500, ct);

        // Change Code based on human feedback
        string code = lastFeedback.Contains("error handling")
            ? "// AI: Improved Logging & Error-Handling added\nif(data == null) throw new ArgumentNullException();" 
            : "// AI: Simple Fix for the NullReferenceException\nif(data == null) return;";

        _logger.LogInformation("LLM has generated a potential fix: {PatchedCode}", code);

        return new Solution { PatchedCode = code };
    }
}
```

The agent needs to interact with the outside world, this is what the Github service is for. 

It handles fetching the initial issue details and creating the final Pull Request. 

Whenever the LLM has generated has generated a solution, we use it zo request a human review (think of a comment in a GitHub PR).

If the LLM has been using more than 
a maximum amounts, the issue needs to be escalated to a lead developer.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

using AbsurdSdk.AiSample.Models;

namespace AbsurdSdk.AiSample.Services;

public interface IGitHubService
{
    Task<Issue> GetIssueDetailsAsync(string id, CancellationToken ct);

    Task<string> CreatePullRequestAsync(string id, string code, CancellationToken ct);

    Task RequestHumanReviewAsync(string issueId, Solution proposedFix, string correlationId, CancellationToken ct);

    Task EscalateToSeniorAsync(string id, string reason, CancellationToken ct);
}

public class GitHubService : IGitHubService
{
    private readonly ILogger<GitHubService> _logger;

    public GitHubService(ILogger<GitHubService> logger)
    {
        _logger = logger;
    }

    public async Task<Issue> GetIssueDetailsAsync(string issueId, CancellationToken ct)
    {
        _logger.LogInformation("GitHub: Gets Ticket #{id} details from the Repository...", issueId);

        await Task.Delay(800, ct);

        return new Issue { StackTrace = "NullReferenceException at PaymentGateway.cs:42" };
    }

    public async Task<string> CreatePullRequestAsync(string issueId, string code, CancellationToken ct)
    {
        _logger.LogInformation("GitHub: PR for Issue #{id} has been created...", issueId);

        await Task.Delay(1200, ct);

        return $"https://github.com/company/repo/pull/{new Random().Next(1000, 9999)}";
    }

    public async Task EscalateToSeniorAsync(string id, string reason, CancellationToken ct)
    {
        _logger.LogCritical("ESCALATION to Senior Developer: Issue #{id} - Reason: {reason}", id, reason);

        await Task.Delay(500, ct);
    }

    public async Task RequestHumanReviewAsync(string issueId, Solution proposedFix, string correlationId, CancellationToken ct)
    {
        _logger.LogInformation("ACTION REQUIRED: Solution for Issue #{id} with Correlation-ID {CorrelationId} has been created: {ProposedFix}...", issueId, correlationId, proposedFix.PatchedCode);

        await Task.Delay(1200, ct);
    }
}
```

## The Autonomous Agent Job ##

We define our logic inside an `IJob`. 

The magic is in the `ctx.Step` method: every time a step completes, its result is automatically checkpointed to the Postgres database. 

If the process crashes or is restarted, the framework replays the job. It skips the already completed steps and loads their results directly from the database.

And then instead of blocking a thread with `Task.Delay` or an infinite polling loop, we use `ctx.AwaitEvent` to wait for human interaction. 

This instructs the engine to suspend the workflow state to the database and free up the worker until an external system fires the specific event being awaited.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

using AbsurdSdk.AiSample.Models;
using AbsurdSdk.AiSample.Services;
using AbsurdSdk.Core;
using System.Text.Json.Nodes;

namespace AbsurdSdk.AiSample;

public class AutonomousAgentJob : IJob<AgentTask, AgentResult>
{
    private readonly ILogger<AutonomousAgentJob> _logger;

    private readonly ILlmService _llmService;
    private readonly IGitHubService _gitHubService;
    private readonly ILocalNotificationService _localNotificationService;

    public AutonomousAgentJob(ILogger<AutonomousAgentJob> logger, ILlmService llmService, IGitHubService gitHubService, ILocalNotificationService localNotificationService)
    {
        _logger = logger;
        _llmService = llmService;
        _gitHubService = gitHubService;
        _localNotificationService = localNotificationService;
    }

    public async Task<AgentResult> ExecuteAsync(TaskContext ctx, AgentTask task)
    {
        _logger.LogInformation("Agent starts researching ticket {IssueId}", task.IssueId);

        // Load the Issue Context first, so the LLM has all relevant information
        var bugReport = await ctx.Step("fetch-issue-context", async () =>
            await _gitHubService.GetIssueDetailsAsync(task.IssueId, ctx.CancellationToken));

        bool isApproved = false;
        int attempt = 0;

        string lastFeedback = "Initial Attempt";

        while (!isApproved && attempt < 3)
        {
            attempt++;

            // Generate CorrelationID for the Event
            string correlationId = $"attempt-{attempt}";

            _logger.LogInformation("Attempt {attempt}/3: Generating a fix based on: {feedback}", attempt, lastFeedback);

            Solution proposedFix = await ctx.Step($"generate-code-fix-{attempt}", async () =>
                await _llmService.GenerateFixAsync(bugReport.StackTrace, lastFeedback, ctx.CancellationToken));

            await ctx.Step($"notify-reviewer-{attempt}", async () => {
                // Notify the reviewer via GitHub service, which could post a link to a GitHub issue or PR for review
                await _gitHubService.RequestHumanReviewAsync(task.IssueId, proposedFix, correlationId, ctx.CancellationToken);
                // Notify the reviewer via local notification service
                await _localNotificationService.NotifyReviewerAsync(task.IssueId, correlationId, ctx.CancellationToken);
            });

            _logger.LogInformation("Review for {CorrelationId} has been requested. Agent goes idle and waits for the code review...", correlationId);

            // Wair for a human decision without blocking a thread
            JsonNode? review = await ctx.AwaitEvent(
                eventName: $"agent-approval:{task.IssueId}:{correlationId}",
                stepName: $"wait-for-human-review-{attempt}"
            );
            
            isApproved = review["approved"]?.GetValue<bool>() ?? false;
            lastFeedback = review["reason"]?.GetValue<string>() ?? "No feedback has been given";

            if (!isApproved)
            {
                _logger.LogWarning("Attempt {attempt} has been rejected: {reason}", attempt, lastFeedback);
            }
        }

        if (isApproved)
        {
            _logger.LogInformation("Fix approved. Creating Pull Request...");

            string prUrl = await ctx.Step("create-pull-request", async () =>
            {
                return await _gitHubService.CreatePullRequestAsync(task.IssueId, "apply-fix", ctx.CancellationToken);
            });

            _logger.LogInformation("Mission accomplished, the PR has been created: {Url}", prUrl);

            return new AgentResult { Success = true, PullRequestUrl = prUrl };
        }
        else
        {
            _logger.LogError("Maximum number of attempts reached. Escalates ticket {IssueId} to a human.", task.IssueId);

            await ctx.Step("notify-senior-developer", async () =>
            {
                await _gitHubService.EscalateToSeniorAsync(task.IssueId, "Agent didn't find a solution after 3 attempts.", ctx.CancellationToken);
            });

            return new AgentResult { Success = false, Reason = "Escalated to human supervisor after 3 failures." };
        }
    }
}
```

## Putting It All Together ##

What's left is registering all dependencies.

In the example I have used TestContainers to spin up a Postgres instance. 

The Connection String for this instance is then passed to the 
`IServiceCollection#AddAbsurdSdk(string connectionString)` Extension method, that registers and wires up all dependencies for interacting 
with the Postgres database.

We then configure a background worker that polls a queue for tasks and maps them to our `AutonomousAgentJob`. 

There are two HTTP endpoints for interacting with the Job: 

* The `/agent/start` endpoint kicks off the process asynchronously and returns immediately.
* The `/agent/review/...` endpoint acts as our callback webhook. 
    * When the human reviewer approves or rejects a fix, this endpoint emits the event back into the queue, which then wakes up the sleeping job right where it left off.

It looks like this.

```csharp
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

using AbsurdSdk;
using AbsurdSdk.AiSample;
using AbsurdSdk.AiSample.Docker;
using AbsurdSdk.AiSample.Models;
using AbsurdSdk.AiSample.Services;
using AbsurdSdk.Core;
using AbsurdSdk.Extensions;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// Start Docker Containers for dependencies
await DockerContainers.StartAllContainersAsync();

string connectionString = $"Host=127.0.0.1;Port=5432;Database=abdurd_db;Username=postgres;Password=password;";

// Add Logging
builder.Services.AddLogging(loggingBuilder => loggingBuilder.AddConsole());

builder.Services.AddSingleton<ILlmService, LlmService>();
builder.Services.AddSingleton<IGitHubService, GitHubService>();
builder.Services.AddSingleton<ILocalNotificationService, LocalNotificationService>();

// Register the Absurd SDK
builder.Services.AddAbsurdSdk(connectionString);

// Configure Workers and Jobs. In this example, we have a queue for AI agents that process tasks related to bug fixing. The
// worker is configured to handle one task at a time and poll for new tasks every second. The job "solve-bug" is defined
// with a maximum of 3 attempts for each task.
builder.Services.AddAbsurdWorker("ai-agent-queue", worker =>
{
    worker
        .SetConcurrency(1)
        .SetPollInterval(1);

    worker.AddJob<AutonomousAgentJob, AgentTask, AgentResult>("solve-bug", options =>
    {
        options.WithMaxAttempts(3);
    });
});

var app = builder.Build();

// A Webhook triggers the Agent, such as a new JIRA ticket or GitHub issue
app.MapPost("/agent/start", async (IAbsurd client, [FromBody] AgentTask task, CancellationToken ct) =>
{
    var result = await client.SpawnAsync(new SpawnOptions
    {
        Queue = "ai-agent-queue"
    }, "solve-bug", task, ct);

    return Results.Ok(new { RunId = result.RunId, Status = $"Agent dispatched to fix Isse #{task.IssueId}" });
});

// A Lead-Developer clicks on "Approve" or "Reject", with Feeedback
app.MapPost("/agent/review/{issueId}/{correlationId}", async (
    IEventPublisher publisher,
    string issueId,
    string correlationId,
    [FromBody] HumanApproval approval,
    CancellationToken ct) =>
{
    // Wake up the agent, that is working on the ticket
    await publisher.EmitEventAsync(queue: "ai-agent-queue", eventName: $"agent-approval:{issueId}:{correlationId}", payload: approval, ct);

    string message = approval.Approved
        ? $"Fix for {correlationId} approved. Agent is now completing its work."
        : $"Fix for {correlationId} rejected. Agent tries again with feedback: '{approval.Reason}'";

    return Results.Ok(new { Message = message });
});

app.Run();
```

## An Example Session with the AI Agent Job ##

After starting the Backend we can see the Postgres container being booted and the Absurd Postgres queue being created:

```
[testcontainers.org 00:00:02.01] Wait for Docker container 5235c226fe2d to complete readiness checks
[testcontainers.org 00:00:03.05] Docker container 5235c226fe2d ready
info: AbsurdSdk.Workers.AbsurdGenericWorker[0]
      Create Queue if not exists: 'ai-agent-queue'
```

In the `AbsurdSdk.AiSample.http` we are now executing a Request for fixing an Issue (say `12345`):

```
### Start the Agent Job for an AI fix for the Issue.
POST https://localhost:5000/agent/start
Content-Type: application/json
{
    "issue_id": "12345"
}
```

It returns the following content:

```
{
  "runId": "019f2db5-d7be-7d89-8fcb-d649e70e698a",
  "status": "Agent dispatched to fix issue #12345"
}
```

In the Console of the Service, you can see the Agent working and requesting feedback. 

```
info: AbsurdSdk.AiSample.AutonomousAgentJob[0]
      Agent starts researching ticket 12345
info: AbsurdSdk.AiSample.Services.GitHubService[0]
      GitHub: Gets Ticket #12345 details from the Repository...
info: AbsurdSdk.AiSample.AutonomousAgentJob[0]
      Attempt 1/3: Generating a fix based on: Initial Attempt
info: AbsurdSdk.AiSample.Services.LlmService[0]
      Agent is thinking: 'Learned from feedback: Initial Attempt'
info: AbsurdSdk.AiSample.Services.LlmService[0]
      LLM has generated a potential fix: // AI: Simple Fix for the NullReferenceException
if(data == null) return;
info: AbsurdSdk.AiSample.Services.GitHubService[0]
      ACTION REQUIRED: Solution for Issue #12345 with Correlation-ID attempt-1 has been created: // AI: Simple Fix for the NullReferenceException
if(data == null) return;...
info: AbsurdSdk.AiSample.AutonomousAgentJob[0]
      Review for attempt-1 has been requested. Agent goes idle and waits for the code review...
```

But let's say we don't like the fix and we want it to be rewritten. 

We will reject the code and tell it to make it simpler:

```
### We cannot approve such a simple fix, reject and tell it to improve.
POST https://localhost:5000/agent/review/12345/attempt-1
Content-Type: application/json
{
    "approved": false,
    "reason": "This is way too simple, add a better error handling strategy!"
}
```

We can then see the Agent doing its work:

```
warn: AbsurdSdk.AiSample.AutonomousAgentJob[0]
      Attempt 1 has been rejected: This is way too simple, add a better error handling strategy!
info: AbsurdSdk.AiSample.AutonomousAgentJob[0]
      Attempt 2/3: Generating a fix based on: This is way too simple, add a better error handling strategy!
info: AbsurdSdk.AiSample.Services.LlmService[0]
      Agent is thinking: 'Learned from feedback: This is way too simple, add a better error handling strategy!'
info: AbsurdSdk.AiSample.Services.LlmService[0]
      LLM has generated a potential fix: // AI: Improved Logging & Error-Handling added
if(data == null) throw new ArgumentNullException();
info: AbsurdSdk.AiSample.Services.GitHubService[0]
      ACTION REQUIRED: Solution for Issue #12345 with Correlation-ID attempt-2 has been created: // AI: Improved Logging & Error-Handling added
if(data == null) throw new ArgumentNullException();...
info: AbsurdSdk.AiSample.AutonomousAgentJob[0]
      Review for attempt-2 has been requested. Agent goes idle and waits for the code review...
```

This looks ok, so let's approve it:

```
### Send Human Feedback to the Agent
POST https://localhost:5000/agent/review/12345/attempt-2
Content-Type: application/json
{
    "approved": true,
    "reason": "Now, this looks good!"
}
```

And in the Console, we can see the PR finally being created:

```
info: AbsurdSdk.AiSample.AutonomousAgentJob[0]
      Fix approved. Creating Pull Request...
info: AbsurdSdk.AiSample.Services.GitHubService[0]
      GitHub: PR for Issue #12345 has been created...
info: AbsurdSdk.AiSample.AutonomousAgentJob[0]
      Mission accomplished, the PR has been created: https://github.com/company/repo/pull/4232
```

And that's it! 👏
