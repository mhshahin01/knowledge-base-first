# AI Architecture Fundamentals: LLM, Agents, Workflows, and the Harness

A plain-language tutorial covering when to use each AI building block, what a harness is, and a real architecture decision walkthrough.

---

## Table of Contents

1. [The Core Question](#the-core-question)
2. [LLM Only](#1-llm-only)
3. [AI Agents](#2-ai-agents)
4. [AI Workflows](#3-ai-workflows)
5. [Decision Map](#decision-map)
6. [What is a Harness?](#what-is-a-harness)
7. [Case Study: Resume Reviewer Portal](#case-study-resume-reviewer-portal)

---

## The Core Question

Every AI system answers one fundamental question:

> **How much autonomy and coordination does this task require?**

The three tiers (LLM, Agent, Workflow) exist on a spectrum from "one smart response" to "self-directed multi-step execution." Choosing the wrong tier wastes money, adds unpredictability, or simply fails to complete the task.

---

## 1. LLM Only

### What it is

A single prompt goes in, a single response comes out. No tools, no loops, no memory between calls.

The LLM is a stateless function:

```
f(prompt) -> text
```

Each call is completely independent from the last.

### When to use it

Use LLM-only when:

- The answer fits in one response
- No external data is needed (no file reading, no web requests)
- The task is self-contained
- You need reasoning or generation, not action

### Real examples

- "Extract all dates from this contract"
- "Rewrite this paragraph in formal English"
- "Is this code vulnerable to SQL injection?"
- "Translate this sentence to Arabic"

### Why it sometimes fails

LLM alone cannot:
- Remember state between turns
- Call APIs or read files
- Track decisions made earlier in a session
- "Apply" changes to a document

---

## 2. AI Agents

### What it is

An LLM running in a loop. The agent can use tools, observe results, and decide the next step. It keeps going until it reaches a goal or gets stuck.

Agents are built on the **ReAct pattern** (Reason + Act):

```
Think
  -> Act (call a tool)
    -> Observe the result
      -> Think again
        -> Act again
          -> ...until done
```

The key property is **dynamic decision-making**. The agent does not know the full execution path upfront. It adapts based on what it discovers.

For a deeper explanation of how agents choose their steps and tools, see Anthropic's [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents).

### When to use it

Use an agent when:

- The steps to solve the problem are not known in advance
- The task requires exploration (search, read files, check results, adjust)
- A failure in one step should change what happens next
- You need a goal achieved, not a procedure followed

### Real examples

- "Debug why this test is failing" (agent reads logs, traces code, runs tests, investigates)
- "Research competitor pricing and summarize" (agent browses, reads, synthesizes)
- "Fix the bug in this repo" (agent reads code, edits, runs tests, re-edits if needed)

### The trade-off

Agents are powerful but non-deterministic. The path taken can vary, costs are harder to predict, and they can go down wrong paths. Do not use an agent when you already know the steps.

---

## 3. AI Workflows

### What it is

Deterministic orchestration of multiple agents or LLM calls. You define the structure. Each agent does its scoped job.

Workflows apply pipeline thinking to AI. You break a complex problem into phases, run them in a defined order (or in parallel), and collect structured results.

```
Phase 1: [Agent A reviews bugs] + [Agent B reviews security]  <- parallel
Phase 2: [Agent C verifies findings from A and B]             <- sequential
Phase 3: [Produce final report]
```

Notice that Agent B's work starts at the same time as Agent A, not after. That is the efficiency gain from parallel phases.

### When to use it

Use a workflow when:

- The problem is too large for one agent's context window
- You can decompose the task into independent parallel dimensions
- You want reproducibility (same structure every time)
- You need specialized agents per domain
- Token budget matters and you want isolated, focused contexts

### Real examples

- Code review across security, performance, and correctness simultaneously
- "Migrate all 20 services" with one worker per service running in parallel
- "Analyze data, write report, publish to Notion" as three sequential phases

### The trade-off

Workflows require upfront design. They are less flexible than agents but more predictable and scalable.

---

## Decision Map

Use this to pick the right tier quickly:

```
Single question, no tools needed?
  -> LLM only

Multi-step problem, but the path is unknown?
  -> Agent (let it figure out the steps)

Multi-step problem, structure is known + parallelism helps?
  -> Workflow (orchestrate specialized agents)
```

If you are unsure between Agent and Workflow, ask yourself:
"Would a new developer be able to draw the execution steps on a whiteboard before running it?"

- Yes: use a Workflow
- No: use an Agent

---

## What is a Harness?

You will see this word in coding agents and agent frameworks. Here is what it means.

A **harness** is the runtime infrastructure that wraps an LLM and gives it the power to act. It is everything around the model that is not the model itself.

Think of it like a pilot and a plane. The pilot (the model) makes decisions. The plane (harness) is the physical infrastructure that makes those decisions real.

### Why a harness? What problems does it solve?

An LLM can suggest an action, but its response cannot execute a tool, carry state into the next call, or enforce a permission boundary. A harness provides the runtime controls that let an agent act over multiple steps and lets the application observe what happened. A one-shot LLM feature may need only a thin wrapper; the need for a fuller harness grows when the model can choose tools and continue working.

| Problem | How the harness helps |
|---|---|
| A model asks to read a file or call an API, but text alone cannot do it. | Runs an allowed tool call and returns the actual result to the model, so the next decision uses an observation rather than a guess. |
| Each model call is separate, so a multi-step task can lose its place. | Maintains the run's messages and tool results; when configured for persistence, it can resume a paused session. |
| The model might request an action it should not take. | Exposes only configured tools, checks permissions, and can stop or reject a disallowed action. |
| Calls can fail, take too long, or consume more resources than expected. | Tracks run status and usage, and applies configured timeouts, retry limits, and stop conditions. |

In the resume reviewer example below, the harness can let the challenge agent read the current resume and return its assessment while withholding the tool that saves edits. The workflow still decides whether that assessment applies to the current resume version.

### If I use Pydantic AI, do I already have a harness?

**Yes, for a basic agent.** Pydantic AI's `Agent` includes a light harness: it runs the model/tool loop, executes tools you register, keeps the messages for a run, and validates structured output. You do not have to build that loop yourself. [Pydantic's harness overview](https://pydantic.dev/docs/ai/harness/) makes this distinction explicit.

Creating an `Agent` does not automatically give it file or shell tools, approval gates, or memory across separate sessions. You choose and scope the tools, configure controls such as usage limits, and store conversation history or add durable execution when your application needs them. See Pydantic's documentation on [agents](https://pydantic.dev/docs/ai/core-concepts/agent/), [message history](https://pydantic.dev/docs/ai/core-concepts/message-history/), and [durable execution](https://pydantic.dev/docs/ai/capabilities/durable_execution/overview/).

Pydantic also offers a separate package called **Pydantic AI Harness** (`pydantic-ai-harness`). It adds optional capabilities and prebuilt agent stacks, such as coding and research agents. It is an extension to consider when the light harness is insufficient, not a prerequisite for using `Agent`. [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/).

### How the harness relates to LLM, Agent, and Workflow

The harness belongs on a different axis from the three choices above. Choosing an LLM call, an agent, or a workflow determines **how the task is controlled**. Designing the harness determines **how that choice runs in your application**.

The [main tutorial](001-agentic-ai-basics.md#3-the-core-agent-loop) explains the loop and its building blocks. Here, focus on where the harness fits:

| Architecture | Relationship to the harness | What adding runtime support does not imply |
|---|---|---|
| LLM-only feature | A small application wrapper sends the request and handles the response. It can record usage, validate output, or retry a failed request. Calling this thin wrapper a "harness" is optional terminology. | Logging or retrying a call does not turn the feature into an agent. |
| Agent | The harness makes the agent's decisions executable: it connects the selected model to the available tools and maintains the running session. The agent is the resulting system, including its task instructions and configuration. | Installing a harness does not give every agent every tool or permission that the harness supports. |
| Workflow | The workflow definition supplies the task structure; runtime code carries it out. An agent step may use its own harness inside that workflow, while other steps use ordinary functions or direct model calls. | A workflow does not require one shared agent harness, or even an agent step. |

In common agent terminology, **model + configured harness = running agent** is useful shorthand. It describes composition, not three independent products to install. Model choice and harness choice are separate decisions: compatible harnesses can expose different capabilities to the same model. See [VS Code's explanation of agent harnesses](https://code.visualstudio.com/docs/agents/concepts/agent-harnesses).

Also, "everything around the model" is a broad description, not a deployment rule. A harness can call a separate tool service, sandbox, or session store; those components need not live in the same process. Anthropic's [Managed Agents architecture](https://www.anthropic.com/engineering/managed-agents) explicitly separates the harness, session, and execution environment.

### Workflow logic versus harness mechanics

Both may be implemented in the same framework, which makes the names easy to confuse. Separate them by asking **what would change if the business process changed?**

For the resume reviewer introduced below:

| Design decision | Where it belongs |
|---|---|
| "Re-evaluate only after accepted edits have been applied." | Workflow logic: a dependency specific to the review process. |
| "A paused review must resume with the same review ID and document version." | Application state contract, supported by the runtime's persistence and resume mechanisms. |
| "The challenge agent may inspect this resume but cannot save an edited copy." | Agent configuration, enforced through the harness and tool permissions. |
| "A model request timed out; report or retry it within the configured limit." | Harness mechanics, reusable across many tasks. |

Changing the review phases changes the workflow. Changing how model requests are dispatched changes the harness. A framework can supply both; the distinction is about responsibility, not necessarily separate libraries.

### What the harness does in Claude Code

| Responsibility | What it means in practice |
|---|---|
| Tool execution | When Claude calls Bash or Edit, the harness runs the command. Claude only declares intent via a function call. |
| Hook system | PreToolUse and PostToolUse hooks are shell commands the harness runs, not Claude. |
| Task tracking | Background agents you spawn are tracked by the harness. It re-invokes Claude when they finish. |
| Permission gating | The "allow this command?" prompt you see is enforced by the harness, not the model. |
| Loop scheduling | ScheduleWakeup tells the harness to wake the model in X seconds. The harness owns the timer. |
| Context management | Compression, caching, and message history are managed by the harness. |

### Why this distinction matters

When documentation says "the harness executes these, not Claude," it means Claude cannot directly run shell commands, manage files, or send HTTP requests. It signals intent through structured tool calls. The harness executes them, captures the result, and feeds it back to Claude in the next turn.

This is why hooks (like PreToolUse) are shell commands in settings.json. The harness intercepts the tool call, runs the hook, and either proceeds or blocks based on the result. Claude never sees the hook code.

### Applying the distinction: a disputed resume concern

Suppose the reviewer flags "leadership experience is unclear," and the user challenges it. This is an illustrative design for the portal:

1. **The workflow delegates a scoped task.** It passes concern `C7`, resume version `v3`, and the user's explanation to the challenge agent. It expects a recommendation and supporting evidence back.
2. **The agent works within that scope.** Its LLM may request another resume excerpt or produce a revised assessment. The harness supplies the configured tools and carries the interaction through to a result.
3. **The result returns to the workflow.** "Withdraw concern C7" is an assessment. It does not itself mean "the user accepted an edit" or "the resume was saved." The application records those as separate events.
4. **The application checks whether the result still applies.** If the user uploaded `v4` while the agent was working, the result for `v3` is stale. The workflow must re-evaluate or reconcile it before continuing; a successfully completed agent run does not resolve that product decision.

This gives the harness an explicit contract: return which run finished, for which input version, with which result or error. The workflow interprets that result in the business process. The LLM supplies the assessment within the run.

### Harness self-check

- **You replace the LLM but keep the same workflow and harness. What stays fixed?** The defined phases and configured execution controls. Assessment quality and the agent's choices can change; compatibility still needs checking.
- **The model can describe an edit, but the harness exposes no write tool. Can the agent save it?** No. Generating a proposed change and having a permitted way to apply it are separate capabilities.
- **An agent run finishes successfully against an old resume version. Is the review ready to continue?** Not automatically. Runtime completion and acceptance of a result into the current workflow are separate checks.

---

## Case Study: Resume Reviewer Portal

### The product description

An end user uploads their resume and gets a full evaluation with strengths, weaknesses, and concerns. Two interaction modes:

**Walkthrough mode (human in the loop):** Present one concern at a time. The user can accept, reject, or challenge each one.

**Show all mode:** Present everything at once. The user can apply all, apply some, or discuss some.

Entry point: upload resume + optional job URL + optional job description.

After applying changes, the system double-checks and gives feedback.

### Why LLM Only does not fit

| What LLM alone cannot do | Why it matters here |
|---|---|
| Fetch a URL | The job posting needs to be read |
| Parse a PDF | The resume is a file, not plain text |
| Track state across turns | The walkthrough has 12 concerns; user accepted 5, rejected 3, and is mid-challenge on the 6th. LLM has no memory of that without explicit state management. |
| Apply changes | "Apply" is an action, not a generation |

LLM calls are still used inside the system for evaluation and re-evaluation. They just cannot be the whole system.

### Why a pure agent does not fit

An agent is designed for dynamic path discovery. It figures out what to do next based on what it observes. But the phases here are already known:

```
Parse -> Evaluate -> Present -> Human acts -> Apply -> Re-evaluate
```

That is a pipeline, not a search problem.

More importantly: accept, reject, and challenge are **user decisions**, not AI decisions. A pure agent would make those decisions autonomously. You need guaranteed stopping points where control returns to the human. Workflows enforce those gates. Agents bypass them.

The one exception is the "challenge" sub-flow. When a user says "I disagree because my 3 years at X is equivalent," the AI needs to reason dynamically and converge on a position. That is genuinely agentic: a short conversational loop with no predetermined outcome.

### Why Workflow is the right primary architecture

The execution phases are deterministic and well-defined. Workflow maps directly to them:

```
Phase 1 (parallel, if all inputs provided):
  - Parse PDF resume
  - Fetch and extract job URL
  - Receive job description text

Phase 2 (sequential):
  - LLM structured call -> { strengths[], weaknesses[], concerns[] }

Phase 3 (branches by mode):
  Walkthrough path:
    - Present concern[i]
    - Wait for user: accept / reject / challenge
    - If challenge: run conversational mini-agent, converge, then continue
  Show All path:
    - Present everything
    - User picks: apply all / apply some / discuss some

Phase 4:
  - Apply accepted changes to resume

Phase 5:
  - LLM structured call -> re-evaluate -> delta feedback
```

### The final architecture: a hybrid

```
+--------------------------------------------------+
|           WORKFLOW (orchestrator)                |
|                                                  |
|  [LLM call] Evaluate -> structured output        |
|       |                                          |
|  [Human gate] Choose mode                        |
|       |                                          |
|  Walkthrough path:                               |
|    For each concern:                             |
|      Accept -> mark done                         |
|      Reject -> mark skipped                      |
|      Challenge -> [MINI-AGENT]                   |
|                   conversational loop            |
|                   converge -> accept or reject   |
|       |                                          |
|  [LLM call] Re-evaluate -> delta feedback        |
+--------------------------------------------------+
```

**Summary of each building block and its role:**

| Building block | Role in this system |
|---|---|
| LLM call | Evaluation engine. Structured output. Also used for re-evaluation. |
| Workflow | Overall orchestrator. Manages phases, parallel inputs, human gates. |
| Mini-agent | Embedded only in the challenge sub-flow. Handles the dynamic conversation. |

---

## Quick Reference

| Tier | Path is known? | Human gates needed? | Dynamic reasoning? | Use it when |
|---|---|---|---|---|
| LLM Only | Yes | No | No | Single-turn generation or reasoning |
| Agent | No | Optional | Yes | Open-ended exploration and goal pursuit |
| Workflow | Yes | Yes | No (except sub-flows) | Multi-phase systems with defined structure |
| Harness | N/A | N/A | N/A | Always present; it is the runtime, not a choice |
