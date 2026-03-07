# Payments MCP — Tool Specification
*Written by Aman Jain — Lead PM, Payments & Growth @ MakeMyTrip*

---

## Overview

Payments MCP is MMT's tool layer that connects the Payments Agent to internal 
payment systems via Model Context Protocol (MCP). Every tool below is called by 
the Payments Agent — not by the user directly.

The Payments Agent is an A2A Server Agent. It receives tasks from the Orchestrator 
Agent via A2A, and internally calls payment systems through MCP tools.

---

## Tool 1: get_payment_methods

**What it does:** Returns available pay modes and UI layer instructions for a user at checkout

**Why it exists:** Payments Agent needs to show contextually relevant options — 
not all saved instruments, only those eligible for this specific transaction. 
It also needs to know which UI component to render — saved card UI, UPI QR, 
wallet UI, or BNPL UI — so the frontend renders the right experience instantly.

**Input:**
- user_id
- order_amount
- merchant_id

**Output:**
- saved_cards[] — only valid, non-expired cards
- upi_ids[]
- wallets[]
- bnpl_eligible (bool) — determined by real-time BNPL partner API call, not RAG
- ui_layer — which UI component to render based on available pay modes 
  and user context (e.g. saved_card_ui / upi_qr_ui / wallet_ui / bnpl_ui)

**Edge Cases:**
1. User has an expired card — must be filtered out silently, never shown at checkout
2. BNPL partner API times out — return bnpl_eligible: false, do not block checkout
3. User has no saved instruments — ui_layer must default to upi_qr_ui, 
   never return empty UI

**Eval:**
- Given a user with 3 saved cards (1 expired), output must contain exactly 2 cards. 
  Expired card appearing = P0 failure — Automated
- Given a user with saved cards + UPI, ui_layer must return saved_card_ui 
  as default — Automated

---

## Tool 2: resolve_paymode

**What it does:** Maps a natural language payment instruction to a specific 
saved payment instrument

**Why it exists:** User says "pay with my HDFC card" or "mere dusre card se kardo" 
— the agent needs deterministic resolution. It cannot guess. If ambiguous, 
it must ask for clarification — never silently resolve to the wrong instrument.

**Input:**
- natural_language_instruction (e.g. "pay with my first card")
- user_id
- available_instruments[]

**Output:**
- resolved_instrument_id
- instrument_type
- last_4_digits
- clarification_required (bool)
- ui_layer — updated UI to render post resolution 
  (e.g. confirm_card_ui / otp_ui)

**Edge Cases:**
1. Ambiguous instruction with multiple matching instruments — must set 
   clarification_required: true, never guess
2. User references a card that doesn't exist — return clear error, 
   do not resolve to closest match

**Eval:**
- Ambiguous input ("pay with my HDFC card" — 2 HDFC cards saved) must trigger 
  clarification_required: true — LLM-as-judge
- Unambiguous input ("pay with HDFC card ending 1234") must resolve correctly 
  in one step — Automated

---

## Tool 3: initiate_cart_mandate

**What it does:** Creates an AP2 cryptographically signed cart mandate 
for the specific transaction

**Why it exists:** Provides cryptographic proof of user intent before any PSP call. 
The mandate contains the exact order details, amount, merchant, and expiry — 
ensuring the agent cannot execute a payment the user did not explicitly approve.

**Input:**
- order_id
- amount
- merchant_id
- user_auth_token
- expiry_minutes (default: 15)

**Output:**
- signed_mandate_object
- mandate_id
- expires_at (timestamp)
- ui_layer — render payment confirmation UI showing order snapshot 
  before user gives final approval

**Edge Cases:**
1. Mandate expires before PSP call is made — agent must detect expiry and 
   re-initiate mandate, never retry with expired mandate
2. Amount in mandate does not match order amount — hard reject, 
   never allow mismatch to reach PSP

**Eval:**
Expired mandate must never reach PSP call. Timestamp check — 
if current_time > expires_at, tool must return expiry error before PSP is called.

---

## Tool 4: process_payment

**What it does:** Executes the payment via PSP using the resolved 
instrument and signed mandate

**Why it exists:** Final execution step. Must be idempotent — 
same request called twice must never result in two charges. 
Every call must be audit logged for liability purposes.

**Input:**
- mandate_id
- instrument_id
- idempotency_key

**Output:**
- payment_id
- status (initiated / pending / success / failed)
- otp_required (bool)
- next_action (e.g. render_otp_ui)
- ui_layer — render OTP UI if otp_required: true, 
  else render payment success UI

**Edge Cases:**
1. Duplicate call with same idempotency_key — must return same payment_id, 
   single charge only. Two charges = P0 incident
2. OTP required — return next_action: render_otp_ui and pause execution. 
   Agent must wait for OTP confirmation before proceeding

**Eval:**
Two identical requests with same idempotency_key must return identical payment_id 
and result in exactly one charge — Automated deduplication test.

---

## Tool 5: check_payment_status

**What it does:** Returns real-time status of a payment in progress

**Why it exists:** Payments Agent streams task status back to Orchestrator Agent 
via A2A. It cannot block on PSP response — it needs to poll status and stream 
updates: initiated → pending → success / failed.

**Input:**
- payment_id

**Output:**
- status (initiated / pending / success / failed)
- amount
- timestamp
- failure_reason (if failed)
- ui_layer — render success UI / failure UI / retry UI based on status

**Edge Cases:**
1. PSP timeout — must return status: pending, never failed. 
   Wrong status here triggers incorrect refund flow
2. Invalid payment_id — return clear error with message, 
   do not return empty response

**Eval:**
PSP timeout scenario must always return pending, never failed — 
Automated. Wrong status on timeout = incorrect refund trigger = P0.

---

## How the Tools Fit Together — End to End Flow
```
User: "Book flight to Goa and pay"

1. Orchestrator Agent → discovers Payments Agent via A2A Agent Card
2. Orchestrator delegates payment task via A2A
3. Payments Agent calls get_payment_methods via MCP
   → returns pay modes + ui_layer (e.g. saved_card_ui)
   → user sees their saved cards
4. User says "pay with my HDFC card" 
   → resolve_paymode via MCP
   → if ambiguous → clarification asked → user confirms
   → ui_layer updated to confirm_card_ui
5. initiate_cart_mandate via MCP
   → AP2 signed mandate created (15 min expiry)
   → ui_layer: render order confirmation screen
6. process_payment via MCP
   → PSP called
   → otp_required: true → ui_layer: render_otp_ui
7. User enters OTP → payment success
   → ui_layer: render success UI
8. check_payment_status via MCP → status: success
9. A2A task state: completed → Artifact returned to Orchestrator
10. Orchestrator confirms booking to user
```

---

## Key Design Principles

1. **Idempotency everywhere** — every tool that touches money must be idempotent
2. **Never guess on paymode** — ambiguity must always trigger clarification
3. **Mandate before PSP** — AP2 signed mandate is non-negotiable before payment execution
4. **UI layer at every step** — every tool returns ui_layer so the frontend always 
   knows what to render without additional calls
5. **Audit trail at every step** — MMT holds liability from user perspective 
   regardless of where failure occurs
6. **Expired = re-initiate, never retry** — expired mandates must never reach PSP

---

*Written by Aman Jain — Lead PM, Payments & Growth @ MakeMyTrip*  
*Last updated: March 2026*
