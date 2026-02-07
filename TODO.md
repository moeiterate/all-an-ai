# TODO

## Mo's Tasks

- [x] Configure ElevenLabs agent
  - [x] Create agent in ElevenLabs dashboard
  - [x] Set voice model and parameters
  - [x] Configure API keys and endpoints
- [x] Set up phone number (ElevenLabs for V1)
  - [x] Get number via ElevenLabs Deploy → Phone Numbers
  - [x] Assign agent to number
  - [x] Test call — agent answers and responds
- [ ] ~~Twilio~~ (skipped for V1; can add later for own number)
- [ ] Validate end-to-end call quality
  - [x] Test call from phone
  - [ ] Verify audio quality
  - [ ] Check response latency
  - [ ] Tune as needed
- [ ] Create repo and commit initial files
  - Initialize git repository
  - Commit scaffolded files
  - Push to GitHub

## Ahmad's Handoff Items

- [x] Define agent purpose and behavior
  - [x] What should the agent do?
  - [x] What tone should it have?
  - [x] What topics should it cover?
- [x] Write system prompt and guardrails
  - [x] Core system prompt (see `prompts/system.md`)
  - [x] Refusal behavior
  - [x] Safety guardrails
- [ ] Optional landing page copy
  - Update `landing/index.html` with final content
  - Add phone number when available

## Done Means

- [x] Phone number is live and accepting calls
- [x] AI agent responds naturally to caller questions
- [ ] Audio quality is clear and latency is acceptable
- [x] System prompt is finalized and deployed
- [ ] Landing page (if used) displays correct phone number
