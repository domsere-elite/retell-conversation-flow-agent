# EPM Inbound - Erik (Conversation Flow Agent)

Retell AI conversation flow agent for Elite Portfolio Management inbound debt collection calls.

## IDs

| Resource | ID |
|----------|-----|
| Agent | `agent_cf676ce4e6044c864f4ecd7e4b` |
| Conversation Flow | `conversation_flow_386d055c83ef` |
| Original LLM Agent (reference) | `agent_64d0e78c207dfaf1d712ae5570` |

## Architecture

This agent uses Retell AI's **conversation flow** (node-based) architecture instead of the older LLM-state architecture. The flow consists of 15 nodes:

```
Greeting (conversation)
  -> Auto Lookup by Phone (function)
    -> Check Account Status (branch)
      -> Verify Identity (conversation)
        -> Deliver Compliance (function)
          -> Opening Frame (conversation)
            -> Payment Waterfall (subagent)
              -> Payment Capture (subagent)
                -> Closing (end)
      -> Manual Account Lookup (subagent)
      -> Transfer to Specialist (transfer_call)
      -> Transfer - Attorney (transfer_call)
  -> Log Dispute (function) -> Transfer
  -> Log Cease & Desist (function) -> Transfer
  -> End - Refused (end)
```

## Files

- `create_conversation_flow.json` - Initial flow creation payload (v0)
- `update_flow_fix.json` - Updated nodes with fixes for attorney override, dispute/C&D routing
- `test-results/final_results.json` - Batch test results (6/6 passing)

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

All 6 core scenarios passing:

| Scenario | Status |
|----------|--------|
| Happy Path - Identity Verified | PASS |
| Attorney Mention - Immediate Transfer | PASS |
| Dispute Declaration | PASS |
| Cease and Desist Request | PASS |
| Wrong Person - Identity Mismatch | PASS |
| Waterfall - Declines Through Hardship | PASS |
