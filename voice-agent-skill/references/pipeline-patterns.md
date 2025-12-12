# Pipeline Patterns Reference

Complete voice agent pipeline patterns including WebSocket servers and async orchestration.

## Full WebSocket Server

```python
import asyncio
import json
import os
from contextlib import asynccontextmanager
from typing import AsyncIterator
from uuid import uuid4

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage
from langchain_core.runnables import RunnableGenerator
from langgraph.checkpoint.memory import InMemorySaver

# Import STT and TTS clients (from other reference files)
from assemblyai_stt import AssemblyAISTT, stt_stream
from elevenlabs_tts import ElevenLabsTTS, tts_stream

# ============================================================================
# Event Types
# ============================================================================

class VoiceEvent:
    """Base voice pipeline event."""
    def __init__(self, type: str, **kwargs):
        self.type = type
        for k, v in kwargs.items():
            setattr(self, k, v)
    
    def to_dict(self) -> dict:
        return {k: v for k, v in self.__dict__.items() if not k.startswith("_")}


# ============================================================================
# Agent Stage
# ============================================================================

def create_voice_agent():
    """Create LangChain agent for voice interactions."""
    
    # Define tools
    def get_menu() -> str:
        """Get the sandwich shop menu."""
        return """
        Sandwiches: BLT ($8), Turkey Club ($10), Veggie Wrap ($9)
        Sides: Chips ($2), Soup ($4)
        Drinks: Soda ($2), Coffee ($3)
        """
    
    def add_to_order(item: str, quantity: int) -> str:
        """Add item to customer's order."""
        return f"Added {quantity}x {item} to your order."
    
    def confirm_order(order_summary: str) -> str:
        """Confirm and submit the order."""
        return f"Order confirmed: {order_summary}. Your order number is #{uuid4().hex[:6].upper()}."
    
    return create_agent(
        model="anthropic:claude-haiku-4-5",
        tools=[get_menu, add_to_order, confirm_order],
        system_prompt="""You are a friendly sandwich shop assistant.
        Help customers browse the menu and place orders.
        Keep responses brief and natural for voice conversation.
        Do NOT use emojis, markdown, or special characters.
        Your responses will be read by text-to-speech.""",
        checkpointer=InMemorySaver(),
    )


async def agent_stream(
    event_stream: AsyncIterator[VoiceEvent],
) -> AsyncIterator[VoiceEvent]:
    """
    Process STT events through agent, yield agent responses.
    
    - Passes through all upstream events
    - Invokes agent on stt_output events
    - Yields agent_chunk, tool_call, tool_result events
    """
    agent = create_voice_agent()
    thread_id = str(uuid4())
    
    async for event in event_stream:
        # Pass through all upstream events
        yield event
        
        # Process final transcripts
        if event.type == "stt_output":
            stream = agent.astream(
                {"messages": [HumanMessage(content=event.transcript)]},
                {"configurable": {"thread_id": thread_id}},
                stream_mode="messages",
            )
            
            async for message, metadata in stream:
                # Yield text chunks
                if hasattr(message, "text") and message.text:
                    yield VoiceEvent("agent_chunk", text=message.text)
                
                # Yield tool calls
                if hasattr(message, "tool_calls") and message.tool_calls:
                    for tool_call in message.tool_calls:
                        yield VoiceEvent(
                            "tool_call",
                            tool=tool_call["name"],
                            args=tool_call["args"],
                        )
                
                # Yield tool results
                if message.type == "tool":
                    yield VoiceEvent(
                        "tool_result",
                        tool=message.name,
                        result=message.content,
                    )
            
            # Signal end of agent turn
            yield VoiceEvent("agent_end")


# ============================================================================
# Pipeline Composition
# ============================================================================

def create_pipeline():
    """Create the full voice pipeline."""
    return (
        RunnableGenerator(stt_stream)
        | RunnableGenerator(agent_stream)
        | RunnableGenerator(tts_stream)
    )


# ============================================================================
# WebSocket Server
# ============================================================================

app = FastAPI(title="Voice Agent Server")


@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    """Handle voice agent WebSocket connections."""
    await websocket.accept()
    
    async def audio_stream() -> AsyncIterator[bytes]:
        """Yield audio bytes from WebSocket."""
        try:
            while True:
                data = await websocket.receive_bytes()
                yield data
        except WebSocketDisconnect:
            pass
    
    pipeline = create_pipeline()
    
    try:
        # Transform audio through pipeline
        output_stream = pipeline.atransform(audio_stream())
        
        async for event in output_stream:
            if event.type == "tts_chunk":
                # Send audio bytes back to client
                await websocket.send_bytes(event.audio)
            else:
                # Send other events as JSON for client UI
                await websocket.send_text(json.dumps(event.to_dict()))
    
    except WebSocketDisconnect:
        pass
    finally:
        await websocket.close()


# ============================================================================
# Server Startup
# ============================================================================

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Alternative: No-Framework Pipeline

For simpler deployments without FastAPI.

```python
import asyncio
import websockets
from typing import AsyncIterator

async def voice_agent_handler(websocket):
    """Handle a single voice agent connection."""
    
    async def audio_stream() -> AsyncIterator[bytes]:
        """Read audio from WebSocket."""
        async for message in websocket:
            if isinstance(message, bytes):
                yield message
    
    pipeline = create_pipeline()
    
    async for event in pipeline.atransform(audio_stream()):
        if event.type == "tts_chunk":
            await websocket.send(event.audio)
        else:
            await websocket.send(json.dumps(event.to_dict()))


async def main():
    """Start WebSocket server."""
    async with websockets.serve(voice_agent_handler, "0.0.0.0", 8000):
        await asyncio.Future()  # Run forever


if __name__ == "__main__":
    asyncio.run(main())
```

## Event JSON Format

Events sent to client as JSON:

```json
// STT partial transcript
{"type": "stt_chunk", "transcript": "I want a"}

// STT final transcript
{"type": "stt_output", "transcript": "I want a BLT sandwich please"}

// Agent text chunk
{"type": "agent_chunk", "text": "Great choice!"}

// Tool call
{"type": "tool_call", "tool": "add_to_order", "args": {"item": "BLT", "quantity": 1}}

// Tool result
{"type": "tool_result", "tool": "add_to_order", "result": "Added 1x BLT to your order."}

// Agent turn complete
{"type": "agent_end"}
```

## Async Utilities

### Merge Async Iterators

```python
import asyncio
from typing import AsyncIterator, TypeVar

T = TypeVar("T")

async def merge_async_iters(*iterators: AsyncIterator[T]) -> AsyncIterator[T]:
    """Merge multiple async iterators into one."""
    queue: asyncio.Queue = asyncio.Queue()
    done_count = 0
    total = len(iterators)
    
    async def consume(iterator: AsyncIterator[T]):
        nonlocal done_count
        try:
            async for item in iterator:
                await queue.put(("item", item))
        finally:
            done_count += 1
            if done_count == total:
                await queue.put(("done", None))
    
    tasks = [asyncio.create_task(consume(it)) for it in iterators]
    
    try:
        while True:
            msg_type, item = await queue.get()
            if msg_type == "done":
                break
            yield item
    finally:
        for task in tasks:
            task.cancel()
```

### Async Queue Bridge

```python
class AsyncQueueBridge:
    """Bridge between sync callbacks and async iteration."""
    
    def __init__(self):
        self._queue: asyncio.Queue = asyncio.Queue()
        self._done = False
    
    def put_sync(self, item):
        """Put item from sync callback (thread-safe)."""
        asyncio.get_event_loop().call_soon_threadsafe(
            self._queue.put_nowait, item
        )
    
    def done(self):
        """Signal completion."""
        self._done = True
        asyncio.get_event_loop().call_soon_threadsafe(
            self._queue.put_nowait, None
        )
    
    async def __aiter__(self):
        """Async iterate over items."""
        while True:
            item = await self._queue.get()
            if item is None and self._done:
                break
            yield item
```

## Connection Management

```python
import logging
from dataclasses import dataclass
from typing import Dict, Optional

logger = logging.getLogger(__name__)

@dataclass
class VoiceSession:
    """Voice agent session state."""
    session_id: str
    thread_id: str
    stt_client: Optional[AssemblyAISTT] = None
    tts_client: Optional[ElevenLabsTTS] = None
    
    async def cleanup(self):
        """Clean up session resources."""
        if self.stt_client:
            await self.stt_client.close()
        if self.tts_client:
            await self.tts_client.close()


class SessionManager:
    """Manage voice agent sessions."""
    
    def __init__(self):
        self._sessions: Dict[str, VoiceSession] = {}
    
    def create_session(self, session_id: str) -> VoiceSession:
        """Create new session."""
        session = VoiceSession(
            session_id=session_id,
            thread_id=str(uuid4()),
        )
        self._sessions[session_id] = session
        logger.info(f"Created session: {session_id}")
        return session
    
    def get_session(self, session_id: str) -> Optional[VoiceSession]:
        """Get existing session."""
        return self._sessions.get(session_id)
    
    async def close_session(self, session_id: str):
        """Close and cleanup session."""
        session = self._sessions.pop(session_id, None)
        if session:
            await session.cleanup()
            logger.info(f"Closed session: {session_id}")
    
    async def close_all(self):
        """Close all sessions."""
        for session_id in list(self._sessions.keys()):
            await self.close_session(session_id)


session_manager = SessionManager()
```

## Health Checks

```python
from fastapi import FastAPI
from datetime import datetime

app = FastAPI()

@app.get("/health")
async def health_check():
    """Health check endpoint."""
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "active_sessions": len(session_manager._sessions),
    }

@app.get("/ready")
async def readiness_check():
    """Readiness check - verify external services."""
    checks = {}
    
    # Check STT
    try:
        stt = AssemblyAISTT()
        await stt._ensure_connection()
        await stt.close()
        checks["stt"] = "ok"
    except Exception as e:
        checks["stt"] = f"error: {e}"
    
    # Check TTS
    try:
        tts = ElevenLabsTTS()
        await tts._ensure_connection()
        await tts.close()
        checks["tts"] = "ok"
    except Exception as e:
        checks["tts"] = f"error: {e}"
    
    all_ok = all(v == "ok" for v in checks.values())
    
    return {
        "ready": all_ok,
        "checks": checks,
    }
```

## Docker Deployment

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install audio processing dependencies
RUN apt-get update && apt-get install -y \
    ffmpeg \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Environment variables
ENV PYTHONUNBUFFERED=1

# Run server
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: "3.8"

services:
  voice-agent:
    build: .
    ports:
      - "8000:8000"
    environment:
      - ASSEMBLYAI_API_KEY=${ASSEMBLYAI_API_KEY}
      - ELEVENLABS_API_KEY=${ELEVENLABS_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```
