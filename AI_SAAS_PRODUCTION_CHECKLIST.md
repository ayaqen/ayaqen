# AI SaaS Production Readiness Checklist

> A practical system-design checklist for founders and developers building AI products that need to be useful, secure, affordable, and reliable.

This guide is for solo builders, students, startup teams, and engineers who want to move from an AI demo to a product real users can trust.

## Before You Build

Write down these answers before choosing frameworks, databases, models, or cloud providers:

- **Target user:** Who has the problem?
- **Painful task:** What are they currently struggling to do?
- **Core outcome:** What result must the product deliver?
- **Critical action:** What is the one action that cannot fail?
- **Trust boundary:** What information should the AI never access?
- **Human approval:** Which actions must always be confirmed by a person?

Use this sentence:

```text
For [target user],
this product helps them [complete an outcome]
by [core workflow],
without requiring them to [current painful process].
```

If this statement is unclear, the architecture will probably become unclear too.

---

## 1. Estimate the Workload

You do not need perfect numbers. You need reasonable limits.

```text
Expected daily users:
Peak requests per minute:
AI requests per user:
Average input tokens:
Average output tokens:
Largest uploaded file:
Required response time:
Data retention period:
Maximum monthly infrastructure budget:
```

Estimate AI usage:

```text
Monthly AI cost =
monthly requests
× average tokens per request
× model price per token
```

Also model three cases:

```text
Expected case: normal user activity
Growth case: 10× normal activity
Abuse case: automated or repeated requests
```

Design for the expected case, but know how the system will react to the other two.

---

## 2. Start With a Simple Architecture

A practical first architecture for many AI SaaS products:

```text
┌──────────────┐
│ Web or Mobile│
│    Client    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ App / API    │
│ Server       │
└───┬────┬─────┘
    │    │
    │    └──────────────┐
    ▼                   ▼
┌──────────┐      ┌───────────┐
│PostgreSQL│      │ Job Queue │
└──────────┘      └─────┬─────┘
                        │
                        ▼
                  ┌───────────┐
                  │ AI Worker │
                  └─────┬─────┘
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
         AI Model   File Storage  Search/RAG
```

Start with:

- One application
- One relational database
- One background worker
- One queue
- One model provider
- One deployment pipeline

Split the system only when real usage shows a clear reason.

---

## 3. Design the User Workflow

Document each important workflow.

```text
1. User submits a request.
2. System validates the request.
3. System checks permissions and usage limits.
4. System creates a traceable job.
5. AI processes the request.
6. System validates the AI output.
7. User receives the result.
8. Usage, cost, latency, and errors are recorded.
```

For every step, ask:

- What can fail?
- Can it be retried safely?
- Could it run twice?
- Does the user know what is happening?
- Can the user cancel it?
- Can the system recover without losing data?

---

## 4. Create Stable API Contracts

Every important endpoint should define:

```text
Input schema
Output schema
Authentication requirement
Permission requirement
Rate limit
Timeout
Possible errors
Retry behavior
Idempotency behavior
```

Recommended practices:

- Validate all input on the server.
- Give every request a unique request ID.
- Use structured outputs when possible.
- Return consistent error objects.
- Never expose provider errors directly to users.
- Make payment, creation, email, and deletion requests idempotent.
- Version APIs before making breaking changes.

Example error response:

```json
{
  "error": {
    "code": "AI_PROVIDER_TIMEOUT",
    "message": "The request took too long. Please try again.",
    "request_id": "req_01JXYZ"
  }
}
```

---

## 5. Treat the AI Model as an Unreliable Dependency

The model can return incorrect, incomplete, malformed, unsafe, or inconsistent results.

Build around that reality.

Record the following for every generation:

```text
Model name
Model version
Prompt version
Temperature
Maximum output tokens
Tools available
Retrieved documents
Latency
Input tokens
Output tokens
Estimated cost
```

Before accepting an AI response:

- Validate its schema.
- Check required fields.
- Check allowed values.
- Reject unexpected tool calls.
- Verify citations when grounding is required.
- Apply business rules in code.
- Never rely on the model to enforce permissions.

Define fallback behavior for:

- Provider outages
- Timeouts
- Malformed output
- Model refusals
- Failed quality checks
- Spending limits

Possible fallback chain:

```text
Primary model → smaller fallback model
AI result → deterministic rule
Live generation → cached result
Automatic action → human review
Full feature → limited mode
```

---

## 6. Build Evaluations Before Scaling

Create a small evaluation dataset before changing models or prompts.

Start with 25 to 50 real examples containing:

```text
User input
Expected behavior
Unacceptable behavior
Required facts
Required format
Safety concerns
Quality score
```

Test:

- Accuracy
- Completeness
- Format compliance
- Hallucination rate
- Citation quality
- Tool selection
- Permission handling
- Prompt-injection resistance
- Latency
- Cost per successful result

Do not replace a model because another model feels better during a few manual tests. Run the same evaluation set against both models and compare the results.

---

## 7. Secure the AI and Agent Layer

Treat model input, uploaded files, webpages, emails, retrieved documents, and tool results as untrusted content.

Minimum controls:

- Authenticate every user.
- Check authorization on the server.
- Separate data by user or organization.
- Give tools the minimum permissions required.
- Use short-lived or limited credentials.
- Store secrets in a secret manager.
- Never include secrets in prompts.
- Never let the model decide what a user can access.
- Validate every tool call before execution.
- Record sensitive actions in an audit log.
- Rate-limit expensive or dangerous actions.

Prompt-injection defenses:

- Clearly separate instructions from external content.
- Mark retrieved content as untrusted.
- Do not allow retrieved documents to redefine system rules.
- Allowlist tools and tool arguments.
- Restrict outbound domains.
- Sanitize filenames, URLs, and structured input.
- Require approval before privileged actions.
- Test direct and indirect prompt-injection attacks.

Require human approval before the AI can:

- Send an email or message
- Publish content
- Delete or overwrite data
- Execute code
- Purchase something
- Transfer money
- Change account permissions
- Contact another person
- Make a legal, medical, hiring, or financial decision

---

## 8. Protect User Data

Classify the information the product handles:

```text
Public
Internal
Confidential
Personally identifiable
Financial
Health-related
Authentication data
User-generated content
```

For each category, define:

- Why it is collected
- Where it is stored
- Who can access it
- How long it is retained
- How it is deleted
- Whether it may be sent to an AI provider
- Whether it may be used for training
- Whether it appears in logs

Minimum protections:

- Encrypt data in transit.
- Encrypt sensitive data at rest.
- Collect only what the product needs.
- Remove sensitive data from logs.
- Support account and data deletion.
- Test tenant isolation.
- Back up important data.
- Test restoring the backup.

A backup that has never been restored is only an assumption.

---

## 9. Design for Failure

Every external dependency will eventually become slow, return an error, or become unavailable.

Use:

- Explicit timeouts
- Limited retries
- Exponential backoff
- Random retry jitter
- Circuit breakers
- Background queues
- Dead-letter queues
- Idempotency keys
- Graceful degradation
- Provider fallbacks
- Feature kill switches

Do not retry:

- Invalid requests
- Permission errors
- Payment declines
- Requests that would create duplicate actions
- Permanent provider errors

Long-running AI work should usually run as a background job rather than keeping an HTTP request open.

---

## 10. Make the Product Observable

A production system should answer:

```text
What failed?
Which user was affected?
Which request caused it?
Which model and prompt were used?
Which tool was called?
How much did the request cost?
How long did each step take?
Can the request be reproduced?
```

Record:

- Request ID
- Job ID
- User or tenant ID
- Model and prompt version
- Input and output token counts
- Estimated cost
- Queue time
- Model latency
- Total latency
- Tool calls
- Validation failures
- Provider errors
- User feedback

Track user-focused objectives such as:

```text
Successful task completion rate
Percentage of requests completed under the latency target
Percentage of AI outputs passing validation
Cost per completed user task
Percentage of jobs requiring manual intervention
```

Do not measure only server uptime. Measure whether users successfully completed the task.

---

## 11. Control Cost Before It Becomes a Problem

Set limits at several levels:

```text
Per request
Per user
Per organization
Per feature
Per hour
Per day
Per month
```

Cost controls:

- Set maximum input and output lengths.
- Reject unnecessarily large files.
- Use smaller models for simple work.
- Cache reusable results.
- Batch compatible requests.
- Move non-urgent work to background jobs.
- Stop recursive agent loops.
- Limit tool calls.
- Detect repeated identical requests.
- Alert when spending changes unexpectedly.
- Create a global emergency spending switch.

Track **cost per useful outcome**, not only cost per token.

A cheap request that produces unusable output is still wasted money.

---

## 12. Deploy Changes Safely

A minimum deployment pipeline should run:

```text
Formatting
Linting
Type checking
Unit tests
Integration tests
API contract tests
Database migration checks
AI evaluations
Security checks
Build
Deployment
Smoke tests
```

Before deployment:

- Back up important data.
- Confirm migrations are reversible.
- Save the previous working version.
- Define the rollback condition.
- Check environment variables.
- Confirm monitoring and alerts work.

Prefer small, reversible deployments.

For risky changes:

```text
Internal users
→ small user percentage
→ larger user percentage
→ full release
```

---

## 13. Create an Incident Plan

Write a basic runbook before the first real incident.

```text
How to disable AI generation
How to disable tool execution
How to revoke provider credentials
How to pause background jobs
How to switch model providers
How to restore the database
How to identify affected users
How to notify users
How to roll back a deployment
```

After an incident, record:

- What happened
- User impact
- Detection method
- Timeline
- Root cause
- What reduced the impact
- What increased the impact
- Actions that prevent recurrence

Focus on improving the system, not blaming a person.

---

## 14. Do Not Add Complexity Too Early

### Do not add microservices until:

- Parts of the application need independent scaling.
- Separate teams own separate domains.
- Deployments regularly block each other.
- A clear service boundary exists.

### Do not add a vector database until:

- The product needs semantic retrieval.
- Simple database or text search has been tested.
- A retrieval evaluation dataset exists.
- Retrieval quality can be measured.

### Do not fine-tune a model until:

- The problem is clearly repeatable.
- Prompting and retrieval have been evaluated.
- Enough high-quality examples exist.
- The improvement can be measured.
- The maintenance cost is understood.

### Do not build multiple agents until:

- A single agent with clear tools has been tested.
- The workflow has real parallel roles.
- Each agent has a limited responsibility.
- Agent communication can be traced.
- Loops and duplicated work can be stopped.

### Do not add Kubernetes until:

- The current deployment platform creates a measurable limitation.
- The team can operate Kubernetes safely.
- The operational cost is justified.

Complexity should solve an observed problem, not make the architecture look advanced.

---

## 15. Final Launch Gate

### Product

- [ ] The target user and core outcome are clear.
- [ ] The most important workflow works end to end.
- [ ] Users understand when AI is involved.
- [ ] Failure messages tell users what to do next.

### AI quality

- [ ] A repeatable evaluation dataset exists.
- [ ] Model and prompt versions are recorded.
- [ ] AI output is validated.
- [ ] A fallback behavior exists.
- [ ] Unsafe or low-quality results can be reported.

### Security

- [ ] Authentication and authorization are enforced.
- [ ] User data is isolated.
- [ ] Secrets are stored safely.
- [ ] Tool permissions follow least privilege.
- [ ] Prompt injection has been tested.
- [ ] High-risk actions require approval.
- [ ] Sensitive actions are audited.

### Reliability

- [ ] External calls have timeouts.
- [ ] Retries are limited and safe.
- [ ] Long jobs use a queue.
- [ ] Duplicate execution is prevented.
- [ ] Backups exist.
- [ ] A backup restoration has been tested.
- [ ] A kill switch exists.

### Operations

- [ ] Errors, latency, token usage, and cost are visible.
- [ ] Alerts identify problems that affect users.
- [ ] Deployment rollback is documented.
- [ ] An incident runbook exists.
- [ ] Monthly spending limits are configured.

---

## Default Decision Rule

When choosing between two architectures:

1. Choose the simpler one.
2. Confirm it meets the current requirements.
3. Measure its real limitations.
4. Add complexity only when the measurement justifies it.

The goal is not to build the most advanced architecture.

The goal is to build a system that users can trust.

---

## Credits

Inspired by Vasanth K's **System Design Cheatsheet** and rewritten as an independent AI SaaS production guide.

The original structure was expanded with modern AI evaluation, agent security, reliability, cost-control, observability, and human-approval practices.

Contributions and corrections are welcome.
