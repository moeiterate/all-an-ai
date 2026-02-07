# all-an-AI

A phone-based AI agent that answers calls using ElevenLabs for voice synthesis. V1 uses ElevenLabs’ built-in phone number; Twilio can be added later for your own number or more control.

## Current state

- **Configured:** ElevenLabs agent (system prompt, voice model, LLM). Phone number set up via ElevenLabs Deploy → Phone Numbers and assigned to the agent.
- **Tested:** Called the number; agent answers and responds. Basic flow works.
- **Next:** Validate call quality (audio clarity, latency), create repo, commit initial files, push to GitHub.

## V1 Scope

- One phone number
- One AI agent personality
- No memory between calls
- No multiple personalities or routing
- Basic call handling: answer, speak, respond

## Out of Scope (V1)

- Conversation memory/history
- Multiple phone numbers
- Multiple agent personalities
- Call recording or analytics
- Authentication or user accounts
- Web dashboard or admin panel

## Success Criteria

A caller can:
1. Dial the phone number
2. Speak to the AI agent
3. Receive helpful or entertaining responses
4. Have a natural-feeling conversation

## Roles

- **Mo**: ElevenLabs agent configuration, Twilio integration, end-to-end validation
- **Ahmad**: Agent purpose definition, system prompt writing, optional landing page content
