---
name: voice-agent
description: Build production-ready voice agents using the LangChain Sandwich Architecture. Use when the user requests to create, implement, or build (1) Voice-enabled chatbots or assistants, (2) Real-time speech-to-text (STT) and text-to-speech (TTS) pipelines, (3) Voice-controlled AI applications with LangChain agents, (4) WebSocket-based audio streaming servers, (5) Conversational voice interfaces with tools, (6) Integration with AssemblyAI, ElevenLabs, Cartesia, or other STT/TTS providers, or (7) Any voice-to-voice AI pipeline architecture.
---

# Voice Agent Skill

Build real-time voice agents combining Speech-to-Text (STT), LangChain agents, and Text-to-Speech (TTS) using the "Sandwich Architecture".

## Core Architecture: The Sandwich

```
Audio Input → STT → LangChain Agent → TTS → Audio Output
```

Three distinct stages connected via async streaming:

1. **STT Stage**: Audio → Text transcripts (AssemblyAI, Whisper, Deepgram)
2. **Agent Stage**: Text → Agent response tokens (LangChain/LangGraph)
3. **TTS Stage**: Text → Audio chunks (ElevenLabs, Cartesia, OpenAI TTS)

### Why Sandwich Architecture?

- Full control over each component
- Access to latest text-modality models
- Transparent behavior with clear boundaries
- Sub-700ms latency achievable with streaming

## Event Flow

| Event | Direction | Description |
|-------|-----------|-------------|
| `stt_chunk` | STT → Client | Partial transcription |
| `stt_output` | STT → Agent | Final transcription |
| `agent_chunk` | Agent → TTS | Text chunk from agent |
| `tool_call` | Agent → Client | Tool invocation |
| `tool_result` | Agent → Client | Tool execution result |
| `agent_end` | Agent → TTS | End of agent turn |
| `tts_chunk` | TTS → Client | Audio chunk for playback |

## Quick Start: Minimal Voice Agent

```python
import asyncio
from typing import AsyncIterator
from uuid import uuid4
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage
from langgraph.checkpoint.memory import InMemorySaver

# Event types
class VoiceEvent:
    def __init__(self, type: str, **kwargs):
        self.type = type
        for k, v in kwargs.items():
            setattr(self, k, v)

# Agent setup
agent = create_agent(
    model="anthropic:claude-haiku-4-5",
    tools=[],  # Add your tools here
    system_prompt="You are a helpful voice assistant. Keep responses concise.",
    checkpointer=InMemorySaver(),
)

async def agent_stream(
    event_stream: AsyncIterator[VoiceEvent],
) -> AsyncIterator[VoiceEvent]:
    """Process STT events through agent, yield responses."""
    thread_id = str(uuid4())
    
    async for event in event_stream:
        yield event  # Pass through all events
        
        if event.type == "stt_output":
            stream = agent.astream(
                {"messages": [HumanMessage(content=event.transcript)]},
                {"configurable": {"thread_id": thread_id}},
                stream_mode="messages",
            )
            async for message, _ in stream:
                if message.text:
                    yield VoiceEvent("agent_chunk", text=message.text)
```

## Reference Files

For detailed implementations, see:

- **STT patterns**: See `references/stt-patterns.md` for AssemblyAI, Whisper, Deepgram integration
- **TTS patterns**: See `references/tts-patterns.md` for ElevenLabs, Cartesia, OpenAI TTS integration
- **Pipeline patterns**: See `references/pipeline-patterns.md` for full WebSocket server, async generators, producer-consumer patterns
- **Frontend patterns**: See `references/frontend-patterns.md` for browser audio capture, WebSocket clients, audio playback

## Implementation Workflow

1. **Select providers**: Choose STT (AssemblyAI/Whisper/Deepgram) and TTS (ElevenLabs/Cartesia)
2. **Create event types**: Define event classes for pipeline communication
3. **Implement STT stage**: Producer-consumer pattern for audio streaming
4. **Implement Agent stage**: LangChain agent with streaming responses
5. **Implement TTS stage**: Send text chunks, receive audio
6. **Create server**: WebSocket endpoint to orchestrate pipeline
7. **Build client**: Browser/app for audio capture and playback

## Key Patterns

### Async Generator Transform

Each stage is an async generator that transforms events:

```python
async def stage_stream(
    input_stream: AsyncIterator[VoiceEvent],
) -> AsyncIterator[VoiceEvent]:
    async for event in input_stream:
        yield event  # Pass through
        if event.type == "target_type":
            # Process and yield new events
            yield VoiceEvent("new_type", data=processed)
```

### Producer-Consumer for STT

```python
async def stt_stream(audio_stream: AsyncIterator[bytes]) -> AsyncIterator[VoiceEvent]:
    stt_client = STTClient()
    
    async def send_audio():
        async for chunk in audio_stream:
            await stt_client.send(chunk)
        await stt_client.close()
    
    send_task = asyncio.create_task(send_audio())
    try:
        async for event in stt_client.receive():
            yield event
    finally:
        send_task.cancel()
```

### Pipeline Composition with RunnableGenerator

```python
from langchain_core.runnables import RunnableGenerator

pipeline = (
    RunnableGenerator(stt_stream)
    | RunnableGenerator(agent_stream)
    | RunnableGenerator(tts_stream)
)

# Use in WebSocket handler
async for event in pipeline.atransform(audio_stream):
    if event.type == "tts_chunk":
        await websocket.send_bytes(event.audio)
```

## Dependencies

```txt
# Core
langchain>=0.3.0
langchain-core>=0.3.0
langgraph>=0.2.0

# STT (choose one or more)
assemblyai>=0.30.0
openai-whisper>=20231117
deepgram-sdk>=3.0.0

# TTS (choose one or more)
elevenlabs>=1.0.0
cartesia>=1.0.0

# Server
fastapi>=0.109.0
uvicorn>=0.27.0
websockets>=12.0

# Audio processing
pydub>=0.25.1
numpy>=1.24.0
```

## Environment Variables

```bash
# STT
ASSEMBLYAI_API_KEY=your_key
# or OPENAI_API_KEY for Whisper API

# TTS
ELEVENLABS_API_KEY=your_key
# or CARTESIA_API_KEY

# Agent LLM
ANTHROPIC_API_KEY=your_key
# or OPENAI_API_KEY
```

## When to Use Each Reference

- Setting up speech-to-text? → Read `stt-patterns.md`
- Setting up text-to-speech? → Read `tts-patterns.md`
- Building the full pipeline? → Read `pipeline-patterns.md`
- Creating browser client? → Read `frontend-patterns.md`
