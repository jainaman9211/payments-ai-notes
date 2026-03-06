# Payments AI Notes
*A PM's guide to A2A, AP2, and Payments MCP — written from first principles*

---

## A2A — Agent2Agent Protocol (Google)

A2A is an open source Agent2Agent Protocol developed by Google. The challenge it solves: 
developers use any number of frameworks (LangGraph, CrewAI, custom) to build agents. 
When agents built on different frameworks need to communicate, there is no common standard. 
Google A2A bridges this gap — meaning any agent, regardless of framework, can talk to 
any other agent without rebuilding integrations from scratch.

### Key Characteristics

1. Any agent can be wrapped with A2A protocol with minimum dev effort
2. Every agent has a JSON **Agent Card** at `.well-known/agent.json` — contains agent 
   capabilities, skills, endpoint, and authentication mechanism
3. Agents communicate over widely accepted protocols — HTTP, gRPC etc.
4. Two primary roles: **Client Agent** (discovers and delegates) and **Server Agent** 
   (executes and responds)
5. Client Agent discovers Server Agent via Agent Card, creates a task, and Server Agent 
   responds with a task ID + streams status updates: `initiated → working → completed`
6. Final output is returned as an **Artifact** to the Client Agent

---

## AP2 — Agent Payment Protocol (Google)

AP2 is developed by Google for secure, verifiable payments between AI Agents and Merchants. 
Built with 60+ partners including Adyen, Mastercard, PayPal, and Revolut.

### Key Characteristics

1. AP2 has two core components — **Verifiable Credentials** (proving agent identity) and 
   **Cryptographically Signed Mandates** (proving payment intent)
2. Two types of mandates:
   - **Intent Mandate** — user's pre-authorisation for future payments based on conditions 
     e.g. "allow this agent to book flights under ₹15,000 on my behalf"
   - **Cart Mandate** — sign-off on a specific transaction: exact order, amount, merchant, 
     order snapshot — immediate and binding
3. A Mandate contains: user auth, merchant details, order snapshot, expiry time (~15 min)
4. Mandates are passed to payment gateways and networks — enabling them to identify 
   agent-initiated transactions, assess risk, and detect fraud
5. Payments are rail-agnostic — works across cards, UPI, bank transfers, stablecoins

---

## Payments MCP — How MMT is Building This

MMT's Agentic Payments is a combination of a **Payments Agent** (A2A) and 
**Payments MCP** (tools layer). Here is the complete flow:

### End-to-End Flow

1. **Orchestration via A2A**
   A main Orchestrator Agent handles all user requests. It discovers the Payments Agent 
   via its A2A Agent Card and understands its capabilities. Orchestrator = A2A Client Agent, 
   Payments Agent = A2A Server Agent.

2. **Task Delegation**
   Once the Orchestrator determines the user wants to pay, it creates a task for the 
   Payments Agent. Payments Agent responds with a task ID and begins streaming status: 
   `initiated → waiting for input → success / failed`

3. **Payment Mode Display via MCP**
   Payments Agent calls Payments Core API through MCP — returns available pay modes, 
   user saved cards, and UI layer instructions for which UI to render.

4. **Pay Mode Resolution via LLM**
   User says "pay with my first card" — Payments Agent resolves this through LLM + 
   prompting to map natural language to the correct saved instrument.

5. **Cart Mandate Creation via AP2**
   Once pay mode is selected, a **Cart Mandate** is created — cryptographically signed, 
   valid for 15 minutes, containing order snapshot, merchant details, and amount.

6. **Payment Execution via MCP**
   Payments Agent calls the PSP through MCP for payment processing and OTP UI rendering.

7. **Completion via A2A**
   Once user enters OTP and payment succeeds, success details are streamed back and 
   returned as an **Artifact** to the Orchestrator Agent, which confirms booking to the user.

---

## How It All Fits Together

| Protocol | Layer | Role |
|---|---|---|
| **MCP** | Agent ↔ Tools | Connects agents to APIs, databases, payment systems |
| **A2A** | Agent ↔ Agent | Agents discover, delegate and communicate with each other |
| **AP2** | Agent ↔ Payment Rails | Secure, verifiable, consent-driven payment execution |

> **One line summary:** MCP = Agent to Tools. A2A = Agent to Agent. 
> AP2 = Agent to Payment Rails. Together they form the complete agentic commerce stack.

---

*Written by Aman Jain — Lead PM, Payments & Growth @ MakeMyTrip*  
*Last updated: March 2026*
