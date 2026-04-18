# 🎙️ Bolna Voice AI — Project Architecture & Setup Guide

> **Goal:** Build a real-time voice AI agent where a user sends a voice message and the agent responds back with a voice message — powered by Deepgram for both speech-to-text and text-to-speech.

---

## 📌 Table of Contents

1. [The Big Picture — How It All Flows](#1-the-big-picture--how-it-all-flows)
2. [Agent Manager — The Brain](#2-agent-manager--the-brain)
3. [Agent Types — What Kind of Agent Are You?](#3-agent-types--what-kind-of-agent-are-you)
4. [Helpers — The Utility Belt](#4-helpers--the-utility-belt)
5. [Input Handlers — How Audio Gets IN](#5-input-handlers--how-audio-gets-in)
6. [Memory / Cache — Temporary, Not Persistent](#6-memory--cache--temporary-not-persistent)
7. [Output Handlers — How Audio Gets OUT](#7-output-handlers--how-audio-gets-out)
8. [Transcriber — Voice → Text (STT)](#8-transcriber--voice--text-stt)
9. [Synthesizer — Text → Voice (TTS)](#9-synthesizer--text--voice-tts)
10. [Is Deepgram Alone Enough?](#10-is-deepgram-alone-enough)
11. [Setup & Run Guide](#11-setup--run-guide)

---

## 1. The Big Picture — How It All Flows

Here is the complete voice-to-voice pipeline:

```
User speaks into mic / browser / phone
           │
           │  WebSocket (ws://server/chat/v1/{agent_id})
           ▼
┌─────────────────────────────────────────────────────────┐
│                  BOLNA SERVER (FastAPI)                  │
│                                                         │
│  Input Handler                                          │
│  (receives raw audio bytes from WebSocket)              │
│           │                                             │
│           ▼                                             │
│  TRANSCRIBER (Deepgram STT)                             │
│  Audio → Text via wss://api.deepgram.com/v1/listen      │
│  "Hello, what are your office hours?"                   │
│           │                                             │
│           ▼                                             │
│  AGENT (LLM — GPT-4o / Claude / etc.)                   │
│  Text → Response text (with optional RAG context)       │
│  "Our office is open Monday to Friday, 9am to 5pm."     │
│           │                                             │
│           ▼                                             │
│  SYNTHESIZER (Deepgram TTS)                             │
│  Text → Audio via wss://api.deepgram.com/v1/speak       │
│           │                                             │
│  Output Handler                                         │
│  (streams audio bytes back over WebSocket)              │
└─────────────────────────────────────────────────────────┘
           │
           │  WebSocket (audio chunks in base64)
           ▼
User hears the agent's voice response 🔊
```

**Key insight:** Everything is streaming. The LLM starts generating text → the synthesizer immediately starts converting it to audio → the audio starts streaming to the user. This is what keeps latency low.

---

## 2. Agent Manager — The Brain

📁 `bolna/agent_manager/`

The agent manager is the **orchestrator**. It is the first thing that runs when a WebSocket connection is established. Think of it as the conductor of the entire pipeline.

### Files in agent_manager/

| File | Role |
|---|---|
| `assistant_manager.py` | Entry point — starts the pipeline, runs tasks sequentially |
| `task_manager.py` | The real workhorse — sets up all components and manages the live conversation loop |
| `interruption_manager.py` | Handles when the user speaks while the agent is still talking |
| `voicemail_handler.py` | Detects if the call hit a voicemail and acts accordingly |
| `base_manager.py` | Shared base class |
| `models.py` | Internal data models for latency tracking |

### How It Works — Step by Step

```
WebSocket connection opens
         │
         ▼
AssistantManager.__init__()
  - Stores agent config (from Redis)
  - Stores WebSocket reference
  - Reads the list of "tasks" from config

         │
         ▼
AssistantManager.run()
  - Loops through tasks one by one (usually just 1 task = conversation)
  - For each task → creates a TaskManager

         │
         ▼
TaskManager.__init__()
  - Creates asyncio Queues (audio_queue, llm_queue, synthesizer_queue)
  - Sets up Input Handler  (to receive audio IN)
  - Sets up Output Handler (to send audio OUT)
  - Sets up Transcriber    (STT — Deepgram)
  - Sets up Synthesizer    (TTS — Deepgram)
  - Sets up LLM Agent      (GPT-4o etc.)
  - Sets up InterruptionManager

         │
         ▼
TaskManager.run()
  - Starts the live conversation loop
  - Listens for transcribed text from the transcriber
  - Sends text to the LLM agent
  - Streams LLM output to the synthesizer
  - Synthesizer streams audio back through the output handler
```

### The Queue System (How Components Talk to Each Other)

```
[Input Handler] → audio_queue → [Transcriber]
                                      │
                               transcriber_output_queue
                                      │
                                      ▼
                                  [LLM Agent]
                                      │
                               synthesizer_queue
                                      │
                                      ▼
                               [Synthesizer] → Output Handler → User
```

Each component is an **async task** running concurrently. They communicate through asyncio queues — no component blocks another.

### Interruption Manager

When the user starts speaking while the agent is talking:
1. Deepgram fires a `SpeechStarted` event
2. `InterruptionManager` evaluates: is this a real interruption or accidental noise?
3. If real: sends `{"type": "clear"}` to the output handler → stops audio playback
4. Cancels the current LLM/synthesizer pipeline
5. Starts processing the new user utterance

---

## 3. Agent Types — What Kind of Agent Are You?

📁 `bolna/agent_types/`

There are **7 agent types** in the codebase:

| Agent Type | Config Key | What It Does |
|---|---|---|
| `StreamingContextualAgent` | `simple_llm_agent` | Basic conversational agent — just a system prompt + LLM. Most common starting point. |
| `GraphAgent` | `graph_agent` | Multi-step conversation flows with nodes and edges. Each node = a stage in the conversation. Supports per-node RAG. |
| `KnowledgeBaseAgent` | `knowledgebase_agent` | RAG-powered agent. Queries your vector store before every LLM call. |
| `GraphBasedConversationAgent` | `llm_agent_graph` | Older graph-based flow (legacy, prefer `graph_agent`) |
| `ExtractionContextualAgent` | extraction | Extracts structured data from a conversation (e.g., fill a form) |
| `SummarizationContextualAgent` | summarization | Summarizes the conversation at the end |
| `WebhookAgent` | webhook | Calls an external API/webhook instead of an LLM |

### For Your Project (Voice AI Agent with Knowledge)

- **Start simple:** `simple_llm_agent` — system prompt + GPT-4o. No knowledge base needed.
- **Add knowledge:** Switch to `knowledgebase_agent` and configure `rag_config` to query your docs before answering.
- **Complex flows:** Use `graph_agent` if the conversation has multiple distinct stages (e.g., greeting → issue capture → resolution → goodbye).

---

## 4. Helpers — The Utility Belt

📁 `bolna/helpers/`

These are not part of the pipeline — they are utility modules used by all other components.

| File | What It Does | Used For Your Project? |
|---|---|---|
| `utils.py` | 1000+ lines of utilities: audio conversion, prompt formatting, base64 encoding, latency calc, etc. | ✅ Yes — used internally everywhere |
| `rag_service_client.py` | HTTP client to query the external RAG proxy server | ✅ Yes — if you use knowledgebase agent |
| `conversation_history.py` | Manages the message history (user/assistant turns) for the LLM | ✅ Yes — always active |
| `function_calling_helpers.py` | Handles LLM function/tool calling — triggers API calls when LLM requests them | ✅ Yes — if you use function calling |
| `language_detector.py` | Detects the language the user is speaking and switches prompts accordingly | Optional |
| `logger_config.py` | Structured logging setup | ✅ Always active |
| `analytics_helpers.py` | Tracks cost and duration stats | Optional |
| `mark_event_meta_data.py` | Tracks audio chunk delivery timing for latency measurement | ✅ Active for telephony |
| `observable_variable.py` | Simple pub/sub for internal state changes (e.g., agent hangup triggered) | ✅ Internal use |
| `expression_evaluator.py` | Evaluates conditions on graph agent edges | Only for graph_agent |

---

## 5. Input Handlers — How Audio Gets IN

📁 `bolna/input_handlers/`

Input handlers are responsible for **receiving audio from the user** and putting it into the transcriber queue.

### Available Input Handlers

| Handler | Provider Key | When To Use |
|---|---|---|
| `DefaultInputHandler` | `default` | Web browser clients, desktop apps, quickstart client |
| `TwilioInputHandler` | `twilio` | Phone calls via Twilio |
| `ExotelInputHandler` | `exotel` | Phone calls via Exotel (India) |
| `PlivoInputHandler` | `plivo` | Phone calls via Plivo |
| `VobizInputHandler` | `vobiz` | Phone calls via Vobiz |
| `SipTrunkInputHandler` | `sip-trunk` | Direct SIP/Asterisk connections |

### For Your Project

Use `DefaultInputHandler` — it receives JSON messages over WebSocket:

```json
// Client → Server (audio frame)
{"type": "audio", "data": "<base64_pcm_audio>"}

// Client → Server (text input, for testing)
{"type": "text", "data": "hello what are your office hours"}

// Client → Server (mark ack — confirms audio was played)
{"type": "mark", "name": "<mark_id>"}
```

The handler decodes the base64 audio and puts raw bytes into the transcriber queue.

---

## 6. Memory / Cache — Temporary, Not Persistent

📁 `bolna/memory/cache/`

### What Exists

Only **one cache implementation** exists in the codebase:

**`InmemoryScalarCache`** — a simple Python dict stored in-process memory.

```python
# It's literally just a dictionary
self.data_dict = {}  # "Permanent only during the duration of the call"
```

### What It's Used For

The cache is used by the **synthesizer** to cache TTS audio for repeated phrases:
- If the LLM produces the same sentence twice, it doesn't re-call Deepgram TTS
- It returns the cached audio bytes directly
- This saves latency and API cost

### ⚠️ Important: It Does NOT Persist

> The comment in the code literally says: "Permanent means permanent only during the duration of the call"

- ❌ Refreshing the page = cache gone
- ❌ New call = cache gone
- ❌ Server restart = cache gone
- ✅ Same call session = cache lives

### Conversation History

The **conversation history** (the messages array sent to the LLM) is managed by `ConversationHistory` class in `helpers/conversation_history.py`. This also lives **only in memory** for the duration of the call. It is not persisted to a database.

### Can We Use Mem0 for Persistent Memory?

**Yes, absolutely.** Mem0 is a great fit for this use case. Here's how it would integrate:

```
User: "My name is Vignesh and I prefer Tamil language"
           │
           ▼
    [After the call ends]
           │
           ▼
    mem0.add("User's name is Vignesh. Prefers Tamil.", user_id="phone_number")

    [Next call from same user]
           │
           ▼
    memories = mem0.search("user preferences", user_id="phone_number")
    # Inject into system prompt: "The user's name is Vignesh and prefers Tamil"
           │
           ▼
    Agent greets: "Hello Vignesh! I'll respond in Tamil..."
```

This is **not built yet** in the project — it would be a custom addition.

---

## 7. Output Handlers — How Audio Gets OUT

📁 `bolna/output_handlers/`

Output handlers are responsible for **sending synthesized audio back to the user**.

### Available Output Handlers

| Handler | Provider Key | When To Use |
|---|---|---|
| `DefaultOutputHandler` | `default` | Web browser / WebSocket clients |
| `TwilioOutputHandler` | `twilio` | Phone calls via Twilio |
| `ExotelOutputHandler` | `exotel` | Phone calls via Exotel |
| `PlivoOutputHandler` | `plivo` | Phone calls via Plivo |
| `VobizOutputHandler` | `vobiz` | Phone calls via Vobiz |
| `SipTrunkOutputHandler` | `sip-trunk` | Direct SIP/Asterisk |

### How DefaultOutputHandler Works

```
Synthesized audio chunk arrives (raw PCM bytes)
         │
         ▼
Base64 encode the bytes
         │
         ▼
Send pre-mark: {"type": "mark", "name": "<id>"} — tells client audio is coming
         │
         ▼
Send audio: {"type": "audio", "data": "<base64_audio>"}
         │
         ▼
Send post-mark: {"type": "mark", "name": "<id>"} — client sends this back as ack
         │
         ▼
Client receives ack → server knows audio was played → enables interruption tracking
```

### For Interruptions

When the user speaks mid-response, the output handler receives a `clear` signal:
```python
# Server → Client
{"type": "clear", "data": null}
# Client must immediately stop playing any queued audio
```

---

## 8. Transcriber — Voice → Text (STT)

📁 `bolna/transcriber/`

The transcriber converts raw audio bytes from the user into text that the LLM can process.

### All Available Transcribers

| Transcriber | Provider Key | Language Support | Notes |
|---|---|---|---|
| **DeepgramTranscriber** | `deepgram` | 30+ languages | ⭐ **Default & Recommended** — fastest, best accuracy |
| `AzureTranscriber` | `azure` | 80+ languages | Good for enterprise/Azure stack |
| `GoogleTranscriber` | `google` | 130+ languages | Good multilingual |
| `AssemblyAITranscriber` | `assembly` | EN focused | Good accuracy, slower |
| `GladiaTranscriber` | `gladia` | 100+ languages | Good multilingual alternative |
| `ElevenLabsTranscriber` | `elevenlabs` | EN | New, ElevenLabs-only |
| `SarvamTranscriber` | `sarvam` | Indian languages | Best for Hindi, Tamil, etc. |
| `SmallestTranscriber` | `smallest` | EN | Ultra-low latency |
| `PixaTranscriber` | `pixa` | — | Specialized |

### How Deepgram Transcriber Works (Streaming Mode)

```
Audio bytes → DeepgramTranscriber
                    │
                    ▼
            sender_stream()
         Streams bytes to:
         wss://api.deepgram.com/v1/listen
         ?model=nova-2
         &vad_events=true
         &endpointing=400
         &interim_results=true
                    │
                    ▼
            receiver() — reads back events:

  "SpeechStarted"  → User started speaking (signals interruption system)
  "Results"        → Interim transcript ("hel", "hello", "hello what")
  "UtteranceEnd"   → User finished — final transcript ready → sent to LLM
  "Metadata"       → Duration info for billing
```

### Key Deepgram Parameters

```python
DeepgramTranscriber(
    model="nova-2",          # nova-2 = best accuracy/speed balance
    language="en",           # or "hi", "ta", "te", etc.
    endpointing="400",       # 400ms silence = end of user's turn
    stream=True,             # streaming mode (always True for real-time)
    encoding="linear16",     # PCM for web, mulaw for Twilio phone
    sampling_rate="16000",   # 16kHz for web, 8kHz for phone
    process_interim_results="true",  # for low-latency interruption detection
)
```

---

## 9. Synthesizer — Text → Voice (TTS)

📁 `bolna/synthesizer/`

The synthesizer converts the LLM's text response into audio that is streamed back to the user.

### All Available Synthesizers

| Synthesizer | Provider Key | Voice Quality | Latency | Notes |
|---|---|---|---|---|
| **DeepgramSynthesizer** | `deepgram` | Good | ⚡ Very fast | ⭐ **Recommended for your project** |
| `ElevenlabsSynthesizer` | `elevenlabs` | 🏆 Best | Moderate | Best natural voice, higher cost |
| `CartesiaSynthesizer` | `cartesia` | Excellent | ⚡ Fast | Great alternative |
| `OPENAISynthesizer` | `openai` | Very good | Moderate | GPT-4o voice |
| `AzureSynthesizer` | `azuretts` | Good | Moderate | Good for enterprise |
| `PollySynthesizer` | `polly` | Good | Moderate | AWS Polly |
| `SmallestSynthesizer` | `smallest` | Good | ⚡ Fast | Low latency focus |
| `SarvamSynthesizer` | `sarvam` | Good | Moderate | Indian languages |
| `RimeSynthesizer` | `rime` | Good | Fast | — |
| `PixaSynthesizer` | `pixa` | — | — | Specialized |

### How Deepgram Synthesizer Works (Streaming Mode)

```
LLM generates text chunk: "Our office is open"
                │
                ▼
DeepgramSynthesizer.sender()
   Sends to: wss://api.deepgram.com/v1/speak
   ?encoding=mulaw&sample_rate=8000&model=aura-zeus-en

   {"type": "Speak", "text": "Our office is open"}
   {"type": "Speak", "text": " Monday to Friday."}
   {"type": "Flush"}   ← end of LLM stream

                │
                ▼
DeepgramSynthesizer.receiver()
   Receives raw audio bytes ← streamed back in real-time

                │
                ▼
Output Handler sends audio to user 🔊
```

### Deepgram TTS Voice Models

```
aura-zeus-en      → Male, confident, professional
aura-luna-en      → Female, friendly
aura-stella-en    → Female, warm
aura-athena-en    → Female, British accent
aura-hera-en      → Female, calm
aura-orion-en     → Male, deep
aura-arcas-en     → Male, casual
aura-perseus-en   → Male, formal
```

---

## 10. Is Deepgram Alone Enough?

**Yes — for a simple voice-to-voice agent, Deepgram handles both ends:**

```
User voice → [Deepgram STT] → text → [LLM] → text → [Deepgram TTS] → Agent voice
```

| Requirement | Deepgram Covers It? |
|---|---|
| Real-time speech-to-text | ✅ Yes (nova-2, streaming) |
| Text-to-speech | ✅ Yes (aura-* voices) |
| Interruption detection | ✅ Yes (VAD events, SpeechStarted) |
| Multiple languages | ✅ 30+ languages |
| Phone calls (mulaw 8kHz) | ✅ Yes |
| Web calls (PCM 16kHz) | ✅ Yes |
| What you still need | An **LLM** (GPT-4o, Claude, etc.) for the AI response |

**The only thing Deepgram doesn't provide is the LLM itself.** You still need OpenAI/Anthropic/Groq for that.

---

## 11. Setup & Run Guide

### Step 1 — Prerequisites

```bash
# Python 3.11+
python --version

# Redis (used to store agent configs)
# Install Redis: https://redis.io/download
# Or run via Docker:
docker run -d -p 6379:6379 redis:alpine
```

### Step 2 — Install Dependencies

```bash
cd C:\Users\Vignesh\Documents\bolna
pip install -r requirements.txt

# Also install websockets for the quickstart client
pip install pyaudio sounddevice
```

### Step 3 — Create .env File

```bash
# Copy the sample
cp .env.sample .env
```

Fill in your `.env`:

```env
# === DEEPGRAM ===
DEEPGRAM_AUTH_TOKEN=your_deepgram_api_key_here
DEEPGRAM_HOST=api.deepgram.com
DEEPGRAM_HOST_PROTOCOL=wss

# === LLM ===
OPENAI_API_KEY=your_openai_api_key_here

# === REDIS ===
REDIS_URL=redis://localhost:6379

# === OPTIONAL ===
RAG_SERVER_URL=http://localhost:8000   # Only if using knowledgebase agent
```

### Step 4 — Create Your Agent Config

Save this as `my_agent.json` (or POST it to the API):

```json
{
  "agent_name": "My Voice Assistant",
  "agent_welcome_message": "Hello! How can I help you today?",
  "tasks": [
    {
      "task_type": "conversation",
      "toolchain": {
        "execution": "parallel",
        "pipelines": [["transcriber", "llm", "synthesizer"]]
      },
      "tools_config": {
        "input": { "provider": "default", "format": "wav" },
        "output": { "provider": "default", "format": "wav" },
        "transcriber": {
          "provider": "deepgram",
          "model": "nova-2",
          "language": "en",
          "stream": true,
          "endpointing": 400
        },
        "llm_agent": {
          "agent_flow_type": "streaming",
          "agent_type": "simple_llm_agent",
          "llm_config": {
            "model": "gpt-4o",
            "provider": "openai",
            "max_tokens": 150,
            "temperature": 0.7,
            "agent_flow_type": "streaming"
          }
        },
        "synthesizer": {
          "provider": "deepgram",
          "stream": true,
          "buffer_size": 40,
          "audio_format": "pcm",
          "provider_config": {
            "voice": "aura",
            "voice_id": "zeus",
            "model": "aura-zeus-en"
          }
        }
      },
      "task_config": {
        "hangup_after_silence": 20,
        "optimize_latency": true,
        "use_fillers": false
      }
    }
  ]
}
```

### Step 5 — Start the Bolna Server

```bash
cd C:\Users\Vignesh\Documents\bolna\local_setup
uvicorn quickstart_server:app --host 0.0.0.0 --port 5001 --reload
```

### Step 6 — Register Your Agent

```bash
# POST the agent config to create it and get an agent_id
curl -X POST http://localhost:5001/agent \
  -H "Content-Type: application/json" \
  -d @my_agent.json

# Response: {"agent_id": "abc-123-...", "state": "created"}
```

### Step 7 — Set Agent ID in .env

```env
ASSISTANT_ID=abc-123-...    # the agent_id from Step 6
```

### Step 8 — Run the Quickstart Client (Test It!)

```bash
cd C:\Users\Vignesh\Documents\bolna\local_setup
python quickstart_client.py
```

This opens your **microphone**, connects to the server over WebSocket, streams your voice, and plays back the agent's audio response through your speakers.

### Step 9 — Add a System Prompt

Create `storage/{agent_id}/conversation_details.json`:

```json
{
  "task_0": {
    "system_prompt": "You are a helpful voice assistant for Acme Corp. You are friendly, concise, and professional. Keep your responses short — under 2 sentences — since this is a voice call. The user cannot read long responses.",
    "user": "",
    "assistant": ""
  }
}
```

---

## 📂 Full Folder Reference

```
bolna/
├── agent_manager/          # 🧠 Orchestrates the entire conversation pipeline
│   ├── assistant_manager.py    # Entry point, runs tasks sequentially
│   ├── task_manager.py         # Core pipeline: STT → LLM → TTS (4000+ lines)
│   ├── interruption_manager.py # Handles user interruptions mid-speech
│   └── voicemail_handler.py    # Voicemail detection logic
│
├── agent_types/            # 🤖 Different AI agent personalities/behaviors
│   ├── contextual_conversational_agent.py  # simple_llm_agent (start here)
│   ├── graph_agent.py                      # graph_agent (multi-step flows)
│   ├── knowledgebase_agent.py              # knowledgebase_agent (RAG)
│   ├── extraction_agent.py                 # Extract structured data
│   ├── summarization_agent.py              # Summarize conversations
│   ├── webhook_agent.py                    # Call external APIs
│   └── graph_based_conversational_agent.py # Legacy graph agent
│
├── transcriber/            # 🎙️ Speech-to-Text (Voice → Text)
│   ├── deepgram_transcriber.py   # ⭐ Use this
│   ├── azure_transcriber.py
│   ├── google_transcriber.py
│   └── ... (8 total)
│
├── synthesizer/            # 🔊 Text-to-Speech (Text → Voice)
│   ├── deepgram_synthesizer.py   # ⭐ Use this
│   ├── elevenlabs_synthesizer.py
│   ├── cartesia_synthesizer.py
│   └── ... (10 total)
│
├── input_handlers/         # 📥 Receives audio from user
│   ├── default.py              # WebSocket / browser
│   └── telephony_providers/    # Twilio, Exotel, Plivo, SIP
│
├── output_handlers/        # 📤 Sends audio back to user
│   ├── default.py              # WebSocket / browser
│   └── telephony_providers/    # Twilio, Exotel, Plivo, SIP
│
├── llms/                   # 🧠 LLM integrations (GPT, Claude, Gemini, etc.)
│
├── helpers/                # 🛠️ Shared utilities
│   ├── utils.py                # Audio conversion, prompt formatting, etc.
│   ├── rag_service_client.py   # Talks to the RAG proxy server
│   ├── conversation_history.py # Manages chat message history
│   └── function_calling_helpers.py  # Tool/function call execution
│
├── memory/                 # 💾 In-memory caching (NOT persistent)
│   └── cache/
│       └── inmemory_scalar_cache.py  # Dict-based TTS audio cache
│
├── models.py               # Pydantic models for all configs
├── enums.py                # Provider enums (deepgram, openai, etc.)
└── providers.py            # Maps provider names to classes
```

---

## 🚀 Quick Start Summary

```
1. Install deps:    pip install -r requirements.txt
2. Set .env:        DEEPGRAM_AUTH_TOKEN + OPENAI_API_KEY + REDIS_URL
3. Start Redis:     docker run -d -p 6379:6379 redis:alpine
4. Start server:    uvicorn quickstart_server:app --port 5001
5. Create agent:    POST /agent with your JSON config → get agent_id
6. Set agent ID:    ASSISTANT_ID=<agent_id> in .env
7. Run client:      python local_setup/quickstart_client.py
8. Speak into mic → hear agent respond! 🎉
```
