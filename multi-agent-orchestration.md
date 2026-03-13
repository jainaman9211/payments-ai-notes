# Multi-Agent Systems & Workflow Patterns

> A PM's guide to thinking about, designing, and debugging multi-agent AI systems.

---

## Part 1 — How to Think About Building a Multi-Agent System

Building a multi-agent system follows a 4-step thinking flow:

**Step 1 — Map the Flow**
Identify the distinct steps in your process. Each step becomes one node (agent).
Example: Read Email → Classify Email → Draft Reply → Send Reply

**Step 2 — Identify What Each Step Needs**
For every step, determine what type of operation it is:
- **LLM Call** — understand, decide, generate text
- **Data Call** — fetch from a database, search docs, call an API
- **User Input** — pause and wait for human input before proceeding
- **External Tool** — MCP server, payment gateway, booking system

**Step 3 — Define Shared State**
Decide what data needs to persist across steps. Store raw data only — not formatted text. If you can derive it, don't store it. Every agent reads from and writes to this shared state (the whiteboard).

**Step 4 — Build and Wire the Workflow**
Connect the nodes. Nodes handle their own routing logic — they decide where to go next based on what they see in state.

---

## Part 2 — Workflow Patterns

### 1. Prompt Chaining
Sequential steps where each step's output becomes the next step's input. The chain is predefined and each LLM call processes what the previous one produced.

**Use when:** Task can be broken into clean, verifiable steps where accuracy matters more than speed.

**MMT Payments example:**
> Paymode Selection → Saved Cards → OTP → Payment Processing

Each step depends on the previous. You cannot process payment before OTP, and you cannot show OTP before card selection.

---

### 2. Routing
A classifier node reads the input and routes it to the most appropriate specialised agent. Not all agents run — only the right one for the job.

**Use when:** You have multiple specialised agents and need intelligent traffic direction.

**MMT example:**
> MakeMyTrip's central chatbot Myra receives any user query. If the user asks about hotels, Myra routes to the Hotel Agent — not the Flight Agent, not the Post-Sales Agent. The routing intelligence lives in Myra, the specialists just execute.

---

### 3. Parallelization (Fan Out / Fan In)
Multiple agents run simultaneously on independent subtasks. Results are aggregated at the end. Reduces latency significantly vs running the same tasks sequentially.

**Use when:** Subtasks are independent of each other and can be executed concurrently.

**MMT example:**
> User asks: "Prepare an itinerary for Bali."
> Orchestrator fans out → Flight Agent (prices) + Hotel Agent (stays) + Activities Agent (things to do) all run in parallel → Aggregator combines into one itinerary.

---

### 4. Evaluator-Optimizer
One LLM generates output. A second LLM evaluates it. If quality is below threshold, it loops back to the generator with feedback. Continues until the evaluator is satisfied **or a max retry limit is hit** — the exit condition is critical, otherwise it becomes an infinite loop.

**Use when:** Output quality matters and you want self-correction built in before the response reaches the user.

**Example:**
> Draft Response → Evaluator checks tone, accuracy, completeness → Score below threshold → loop back with feedback → Regenerate → Evaluate again → Pass → Send

---

### 5. Agents (Fully Dynamic)
No predefined path. The LLM decides which tools to call, in what order, based on what it observes. The workflow emerges dynamically.

**Use when:** The problem is open-ended and cannot be decomposed upfront.

**Important:** A real-world agentic system is almost always a **combination of workflows and agents** — some steps are fixed (prompt chaining, routing), some are dynamic (agent decides next step). Pure fully-dynamic agents are rare in production.

---

## Part 3 — What Can Go Wrong at Agent Handoffs

Every time one agent passes to another, things can break. Here are the 6 failure modes:

### 1. Context Loss
Agent A had full context of the user's intent. Agent B only receives what Agent A explicitly passed. If Agent A summarised poorly, Agent B makes wrong decisions.

**Solution:** Shared state (every agent writes outputs to a common whiteboard, every agent reads from it). Also define explicit handoff contracts — Agent A must pass a defined schema to Agent B, not just whatever it decides.

---

### 2. Conflicting Output
Two parallel agents return contradictory results. Without a resolution strategy, the aggregator doesn't know whose answer to trust.

**Solution:** The orchestrator must define a resolution strategy upfront — majority vote, most recent wins, highest confidence score wins. Human-in-the-loop (HITL) is the fallback when no rule can resolve the conflict automatically.

---

### 3. Error / Failure Propagation
Agent A fails silently. Agent B never receives input and gets stuck — or worse, proceeds with no data and produces a wrong answer.

**Solution:** Every handoff needs a defined timeout + fallback state. Agent B should not wait forever. After X seconds, either use a default value or surface the error explicitly. Never fail silently.

---

### 4. High Latency
Each agent adds processing time. In a sequential pipeline of 5 agents at 2 seconds each = 10 second response. Users abandon long before that.

**Solution:** Parallelise where possible. Only chain sequentially when steps are truly dependent on each other.

---

### 5. Deadlock
Agent A is waiting for Agent B. Agent B is waiting for Agent A. The system hangs with no exit.

**Solution:** No agent should call back an agent already in its own call chain. Orchestrators should maintain a call graph and detect cycles before they occur.

---

### 6. Non-Determinism
LLM outputs are never identical twice. Agent B built on the assumption of consistent input from Agent A will behave unpredictably across runs.

**Solution:** Define strict output schemas for every handoff. Agent A's output must match a defined contract — structured, validated, not free-form text. This is the same principle as idempotency in payments.

---

## Summary

| Pattern | Control | Use For |
|---|---|---|
| Prompt Chaining | Fixed | Sequential tasks with verifiable steps |
| Routing | Fixed | Directing traffic to specialised agents |
| Parallelization | Fixed | Independent subtasks, low latency needed |
| Evaluator-Optimizer | Fixed | Quality-sensitive outputs with self-correction |
| Agents | Dynamic | Open-ended tasks that can't be decomposed |

**The rule:** Start with the simplest workflow pattern that solves your problem. Only introduce dynamic agents when fixed workflows genuinely can't handle the complexity.

---

*Written by Aman Jain | March 2026*
*Based on: LangGraph docs, Anthropic's Building Effective Agents, A2A Protocol spec*
