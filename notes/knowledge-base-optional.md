# Knowledge Base - Optional for V1

## Recommendation: Skip for V1

The system prompt should be sufficient for basic diagnostics. Test without a knowledge base first, then add one if you find the agent needs more specific reference material.

## If You Want to Add One (Minimal Approach)

Create **one document** with quick reference info:

### Suggested File: `common-car-problems.md`

Include:
- **Warning light meanings** (check engine, oil pressure, battery, etc.)
- **Common symptoms → likely causes** (won't start → battery/starter/alternator)
- **Basic diagnostic questions** (what to ask for each symptom type)
- **Safety red flags** (when to pull over immediately)
- **Basic troubleshooting steps** (check fluids, fuses, battery connections)

Keep it short - 1-2 pages max. The LLM can reason through most issues; the knowledge base is just for quick reference on specific facts (like what a particular warning light means).

## When to Add Later

Add a knowledge base if you find the agent:
- Gets warning light meanings wrong
- Needs specific make/model troubleshooting
- Should reference repair manuals or technical specs
- Needs to know current recall information

For V1, the system prompt + LLM reasoning should handle most cases.
