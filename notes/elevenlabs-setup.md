# ElevenLabs Setup Guide

## Current State
You're on the Agents dashboard. You need to create a new agent.

## Step 1: Create an Agent

1. Look for a **"Create Agent"** or **"New Agent"** button (usually top-right or in the main content area)
2. Click it to start the agent creation flow
3. Give it a name (e.g., "all-an-AI" or whatever you want)

## Step 2: Configure the Agent

Once created, you'll configure it across these tabs:

### General Tab (you're here)
- **Agent Name**: Already set
- **System Prompt**: Paste the final prompt from `prompts/system.md` (wait for Ahmad or use placeholder for now)
- **First Message**: Optional greeting when call connects

### Audio Tab
- **Voice Selection**: Choose a voice model
- **Voice Settings**: Adjust stability, similarity, style (start with defaults)
- **Response Format**: Set to conversational/phone-friendly

### LLMs Tab
- **Model Selection**: Choose your LLM (GPT-4, Claude, etc.)
- **Temperature**: Controls randomness (0.7-0.9 for conversational)
- **Max Tokens**: Response length limit

### Phone Numbers Tab (Deploy section)
- This is where you'll connect your Twilio number later
- ElevenLabs can provide a phone number OR you can use your Twilio number
- For V1, using ElevenLabs' built-in phone number might be simpler than Twilio integration

## Step 3: Get Phone Number

**Option A: Use ElevenLabs Phone Number (Simpler for V1)**
- Go to **Deploy → Phone Numbers**
- Purchase/configure a number through ElevenLabs
- Assign your agent to that number
- Done - no Twilio needed for V1

**Option B: Use Twilio (More Control)**
- Keep your Twilio setup
- Configure webhook in Twilio to point to ElevenLabs API
- More complex but gives you more control

## Recommendation for V1

Use **Option A** (ElevenLabs phone number) to get started faster. You can always switch to Twilio later if needed.

## Next Steps

1. Create the agent
2. Set a placeholder system prompt (you can update it when Ahmad provides the final one)
3. Choose a voice
4. Get a phone number through ElevenLabs
5. Test by calling the number

## Where to Find Things

- **Agents**: Left sidebar → Configure → Agents
- **Phone Numbers**: Left sidebar → Deploy → Phone Numbers
- **Conversations**: Left sidebar → Monitor → Conversations (to see call logs)
- **Settings**: Left sidebar → Settings (for API keys, billing, etc.)
