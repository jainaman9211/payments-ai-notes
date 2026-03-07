# RAG + Evals Notes
*Written by Aman Jain — Lead PM, Payments & Growth @ MakeMyTrip*

---

## RAG — Retrieval Augmented Generation

### What is RAG and Why it Exists

RAG stands for Retrieval Augmented Generation. It is a method to provide LLM with 
information and data which is specific to your industry or internal to your company, 
so LLMs are able to answer factually correct and domain specific answers to user queries.

For example — a bank has a chatbot with internal policies not available on the internet. 
When a user asks a query, the LLM fetches the answer from the policy document and gives 
a domain specific answer instead of a generic one.

---

### The 5-Step RAG Pipeline

1. **Load** — Load your specific data through a document loader
2. **Chunk** — Divide data into smaller meaningful chunks. LLM context windows are limited 
   and larger inputs increase errors
3. **Embed** — Convert text into mathematical form (vectors) and index using techniques 
   like cosine similarity, Euclidean distance etc.
4. **Store** — Store embedded vectors in a vector database
5. **Retrieve + Generate** — When user query comes, retrieve the right chunk and pass 
   it to LLM to generate the answer

---

### When to Use RAG vs Fine-tuning vs Prompting

| Scenario | Use |
|---|---|
| Private or internal data that changes frequently | RAG |
| Change agent tone, style, or domain behaviour | Fine-tuning |
| LLM already knows enough, simple task | Prompting |
| Real-time or frequently updated information | RAG |

---

### RAG in Payments Agent — When to Use and When Not To

**Use RAG:**
When a user asks *"when will I get my refund?"* — MMT has internal policy documents 
covering all refund cases such as cancellation, no-show, partial cancellation etc. 
RAG retrieves the right policy and LLM answers correctly.

**Do NOT use RAG:**
When a user asks *"am I eligible for BNPL?"* — eligibility is not determined by MMT. 
It is determined by BNPL partners such as Simpl and LazyPay. MMT hits the BNPL Partner 
API with the user's phone number to get eligibility in real time. This is an API call 
through MCP, not a RAG call.

---

### What Can Go Wrong in RAG — Two Failure Modes

**1. Bad Chunking**

- **What it is:** When data is divided incorrectly or incompletely into chunks
- **MMT Example:** Cancellation policy says — if cancelled within 24 hours, user gets 
  100% refund. If cancelled after 24 hours, user gets 50% refund. If these two rules 
  are split into separate chunks, and only Chunk 2 is retrieved, LLM tells the user 
  50% refund even though they cancelled within 24 hours — wrong answer, bad experience
- **Fix as a PM:** Define chunking rules — divide data by topic, not word count. 
  A single policy topic must never be split across chunks

**2. Bad Embedding / Wrong Retrieval**

- **What it is:** When the retriever fetches the wrong document or chunk entirely
- **MMT Example:** User asks about hotel cancellation policy — retriever returns flight 
  cancellation policy because both documents contain similar words like "cancellation", 
  "refund", "policy" and their embeddings are semantically close
- **Fix as a PM:** Test the retriever with real user queries before launch. Build an eval 
  where given a specific query, the correct chunk must be retrieved — pass/fail. 
  Never rely on embeddings alone without retrieval testing

---

## Evals — How to Know When Your AI is Good Enough

### What is an Eval

An eval is your acceptance criteria for an AI feature. Before shipping, you define 
what good output looks like, what bad output looks like, and what must never happen.

---

### 3 Types of Evals

| Type | When to Use | Payments Agent Example |
|---|---|---|
| **Automated** | Deterministic, exact outputs | Expired card must never appear at checkout |
| **LLM-as-judge** | Subjective quality judgment | Was paymode resolution correct? |
| **Human eval** | High stakes, first launch | Is payment confirmation message clear? |

---

### Evals for MMT Payments Agent

**Why Only Paymode Resolution Needs Evals**

Payments Agent does three things:
1. Paymode Resolution
2. Correct UI Rendering
3. Payment Completion

Steps 2 and 3 are fully deterministic — once paymode is resolved, the system 
behaves predictably. Only Step 1 is non-deterministic.

Example: User has 3 saved cards:
- HDFC \*\*1234
- ICICI \*\*3678
- HDFC \*\*5678

User says *"pay with my HDFC card"* — which one? 1234 or 5678? The agent cannot 
guess. It must ask. Or user says in Hindi *"mere dusre card se kardo"* — does that 
mean the ICICI card or the second HDFC card? This ambiguity is what evals must catch.

---

### Scoring Rubric

- **Score 5** — Correct instrument resolved immediately, zero friction
- **Score 4** — Correct instrument resolved but asked one unnecessary clarification
- **Score 3** — Correct resolution only after user gave very specific input
- **Score 2** — Wrong resolution but agent catches itself and asks for clarification
- **Score 1** — Wrong resolution, no error flagged, payment attempted on wrong instrument

---

### Test Set

- **Size:** 30-40 cases
- **Cases to include:** ambiguous instructions, multiple cards of same bank, 
  indirect references ("my second card"), Hindi instructions, wrong card numbers 
  given by user, single card users, expired card in list

---

### 5 Evals for Payments Agent

| Eval | Input | Good Output | Bad Output | Type |
|---|---|---|---|---|
| Pay mode resolution | "Pay with my HDFC card" — 2 HDFC cards saved | Clarification asked | Wrong card selected silently | LLM-as-judge |
| Expired card filtering | User with 1 expired + 2 valid cards | Only 2 cards shown | Expired card shown at checkout | Automated |
| Mandate expiry | PSP call attempted after 15 min | Agent re-initiates mandate | Expired mandate sent to PSP | Automated |
| Hindi instruction | "Mere dusre card se kardo" | Clarification or correct resolution | Wrong card selected | LLM-as-judge |
| Duplicate payment | Same idempotency key called twice | Same payment_id, 1 charge only | Two charges raised | Automated |

---

### Launch Threshold

- **Beta:** 95% of test cases must score 4 or above
- **Hard rule:** Zero Score 1 outputs tolerated — wrong card charged = P0 incident
- **Post launch:** Sample 20 real interactions weekly and run through LLM judge
- **Every prompt change:** Re-run full test suite before deploying to production

---

## Key Takeaway

As a PM, RAG quality = chunking quality + retrieval quality. Both need evals. 
Evals are your acceptance criteria — you don't write the code, but you define 
what correct looks like and what the bar is before shipping.
