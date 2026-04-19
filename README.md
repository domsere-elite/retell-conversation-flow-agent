# EPM Inbound - Erik (Conversation Flow Agent)

Retell AI conversation flow agent for Elite Portfolio Management inbound debt collection calls.

## IDs

| Resource | ID |
|----------|-----|
| Agent | `agent_cf676ce4e6044c864f4ecd7e4b` |
| Conversation Flow | `conversation_flow_386d055c83ef` |
| Original LLM Agent (reference) | `agent_64d0e78c207dfaf1d712ae5570` |

## Architecture

This agent uses Retell AI's **conversation flow** (node-based) architecture instead of the older LLM-state architecture. The flow has 25 nodes as of v47:

```
Greeting (conversation)
  -> Auto Lookup by Phone (function)
    -> Check Account Status (branch)
      -> Verify Identity (conversation)           [strict direct-response rule]
        -> Compliance - Speak Mini Miranda (conversation, static_text)
          -> Deliver Compliance (function, logger only)
            -> Opening Frame (conversation)       [ends with yes/no question]
              -> Step A - Full Balance (conversation)
                -> Step B - Six Payments (conversation)     [decoupled from "today"]
                  -> Step C1 - Settlement 80% (conversation)
                    -> Step C2 - Settlement 70% Final (conversation)
                      -> Step D - Hardship Plan (conversation)  [pay-frequency math]
                        -> Schedule Plan (conversation)
                          -> Plan Recap (conversation)
                            -> Payment Capture (subagent)
                              -> Payment Agreement Disclosure (conversation)
                                -> Closing (end)
      -> Manual Account Lookup (subagent)
      -> Wrong Person (end)
      -> Transfer to Specialist (transfer_call)
      -> Transfer - Attorney (transfer_call)
  -> Log Dispute (function) -> Transfer
  -> Log Cease & Desist (function) -> Transfer
  -> End - All Options Refused (end)
  -> Settled Account Handoff (conversation)
```

The waterfall is split into five state-anchored nodes (Step A -> B -> C1 -> C2 -> D) so the agent cannot regress to an earlier option when the caller pushes back on amount, timing, or uses profanity. Each step node is an independent conversation node; transitions only advance forward.

## Files

- `create_conversation_flow.json` - Current deployed flow (v47 structure, 25 nodes)
- `update_flow_v47.json` - PATCH payload applied in v47 (source of truth for the v47 edits)
- `update_flow_v48.json` - Ready-to-apply payload adding decline-rerun branch (see v48 section; not yet deployed — Retell published-flow lock)
- `test-results/final_results.json` - Batch test results

## v47 Changes (Apr 2026)

Addresses issues observed in v45/v46 test calls:

1. **Waterfall split** into five conversation nodes so state cannot regress to Step A on caller frustration or timing pushback.
2. **Step B decouples "first payment today"** from the 6-monthly plan offer. Timing is handled by a dedicated `Schedule Plan` node so "I don't get paid until X" never collapses the plan.
3. **Step D hardship math** rewritten to speak per-paycheck amounts matching the caller's pay frequency. The agent no longer says "up to 18 months" -- it states the exact number of payments derived from `{{current_balance}} / per_paycheck_amount`.
4. **Step C2** caps "client is strict" repetition at two turns, then advances to Step D instead of looping.
5. **Mini Miranda** is now delivered by a `static_text` conversation node (`node_compliance_speak`) before the logging function, eliminating the risk of the LLM skipping the disclosure.
6. **Verify Identity** adds a direct-response rule: a "yes" only counts as verification if the agent's immediately preceding utterance contained the full name + DOB question.
7. **Opening Frame** closes with a direct yes/no question ("Can we take care of the full balance today?") instead of "Let's go over your options", which was causing dead air.
8. **Global prompt** gains Loop-Breaking, State-Anchoring, and De-escalation rules that forbid the "quickest way" fallback line from being used as a recovery.

## v48 Changes (PENDING DEPLOY)

Adds a branch after compliance so callers whose account lookup returned `status == "decline"` are offered a quick re-run of their last failed payment instead of walking through the full waterfall.

New nodes:
- `node_post_compliance_branch` (branch) — routes on `{{status}} == "decline"`
- `node_decline_rerun` (conversation) — "I see your last payment attempt of {{last_payment_amount}} didn't go through. Would you like me to try running that again now?"
  - YES → `node_payment_capture` (charges exactly `{{last_payment_amount}}`, NOT the full balance)
  - NO → `node_opening_frame` (fall back to normal waterfall)
  - dispute / C&D / attorney → standard routes

The rerun amount is whatever declined before, not the current balance. If backend doesn't populate `last_payment_amount`, the node falls back to the normal waterfall instead of inventing a number.

Redirected edge: `node_deliver_compliance.else_edge` now points to `node_post_compliance_branch` (was `node_opening_frame`). All other verified-identity paths unchanged; Mini-Miranda still delivered before the branch is evaluated.

**Backend requirement:** On account lookup, populate the `last_payment_amount` dynamic variable with the dollar amount of the most recent declined transaction. Also set `status = "decline"` when that applies. `payment_decline_reason` is optional and used for caller-facing explanation if set.

**Deploy requirement:** Retell's public API blocks PATCH on published flows. To apply `update_flow_v48.json`:
1. Open the Retell dashboard for `conversation_flow_386d055c83ef` and click "Edit" (creates an unpublished draft).
2. Run `curl -X PATCH https://api.retellai.com/update-conversation-flow/conversation_flow_386d055c83ef --data-binary @update_flow_v48.json -H "Authorization: Bearer $RETELL_KEY" -H "Content-Type: application/json"`.
3. Call `publish-agent/agent_cf676ce4e6044c864f4ecd7e4b` to lock and activate.

## Agent Settings

- **Voice**: openai-Onyx @ 1.14x speed
- **Model**: GPT-4.1 (temperature 0)
- **DTMF**: Enabled (# termination, 5s timeout)
- **Language**: en-US
- **Webhook**: `https://crm.eliteportmgmt.com/api/voice/webhooks/retell`
- **Transfer number**: +18333814416

## Tools

| Tool | Endpoint | Purpose |
|------|----------|---------|
| autoLookupByPhone | /api/voice/tools/lookup-account | Phone-based account lookup |
| findAccountByPhone | /api/voice/tools/lookup-account | Name+DOB account lookup |
| deliverCompliance | /api/voice/tools/log-compliance | Log Mini Miranda delivery |
| processLivePayment | /api/voice/tools/process-payment | Charge card immediately |
| tokenizeCard | /api/voice/tools/tokenize-card | Save card for arrangements |
| confirmPaymentArrangement | /api/voice/tools/confirm-arrangement | Confirm multi-payment plan |
| logDispute | /api/voice/tools/log-dispute | Log dispute declaration |
| logCeaseAndDesist | /api/voice/tools/log-cnd | Log C&D request |

## Test Results

Regression scenarios to replay after each change (from the v45 transcripts that exposed the bugs fixed in v47):

| Scenario | Desired behavior |
|----------|--------|
| Happy Path - Identity Verified | Accept, Mini Miranda spoken verbatim, reach Closing |
| Attorney Mention - Immediate Transfer | Transfer regardless of step |
| Dispute Declaration | Log dispute, transfer |
| Cease and Desist Request | Log C&D, transfer |
| Wrong Person - Identity Mismatch | Route to Wrong Person end |
| Waterfall - Declines Through Hardship | Advance A -> B -> C1 -> C2 -> D without regression |
| "Yeah" to "you there?" | Re-ask verification -- does NOT advance to compliance |
| Settlement question repeated x4 | Same number every time, no fallback to full balance |
| Insult during Step D | Acknowledge once, stay on Step D |
| "I don't get paid until May 1" after 6-month plan accepted | Schedules first payment May 1, no regression to Step A |
| Biweekly pay on $1500 balance, $100 min | States "15 payments, about 7 months" -- never "up to 18 months" |
| Settlement countered at $1000 on $1500 balance | Step C1 -> C2 -> Step D (no infinite loop) |
