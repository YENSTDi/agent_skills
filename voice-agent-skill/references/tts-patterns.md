# TTS Patterns Reference

Text-to-Speech integration patterns for voice agents.

## ElevenLabs Streaming TTS

Real-time text-to-speech with WebSocket streaming.

```python
import asyncio
import base64
import json
import os
from typing import AsyncIterator
import websockets
from websockets.client import WebSocketClientProtocol

class TTSEvent:
    """TTS audio chunk event."""
    def __init__(self, audio: bytes):
        self.type = "tts_chunk"
        self.audio = audio
    
    @classmethod
    def chunk(cls, audio: bytes) -> "TTSEvent":
        return cls(audio)


class ElevenLabsTTS:
    """ElevenLabs streaming TTS client."""
    
    def __init__(
        self,
        api_key: str | None = None,
        voice_id: str = "21m00Tcm4TlvDq8ikWAM",  # Rachel voice
        model_id: str = "eleven_turbo_v2_5",
        output_format: str = "pcm_16000",
    ):
        self.api_key = api_key or os.getenv("ELEVENLABS_API_KEY")
        self.voice_id = voice_id
        self.model_id = model_id
        self.output_format = output_format
        self._ws: WebSocketClientProtocol | None = None
    
    async def _ensure_connection(self) -> WebSocketClientProtocol:
        """Establish WebSocket connection."""
        if self._ws is None:
            url = (
                f"wss://api.elevenlabs.io/v1/text-to-speech/{self.voice_id}"
                f"/stream-input?model_id={self.model_id}"
                f"&output_format={self.output_format}"
            )
            self._ws = await websockets.connect(url)
            
            # Send initial configuration (BOS - Beginning of Stream)
            bos_message = {
                "text": " ",
                "voice_settings": {
                    "stability": 0.5,
                    "similarity_boost": 0.75,
                },
                "xi_api_key": self.api_key,
            }
            await self._ws.send(json.dumps(bos_message))
        
        return self._ws
    
    async def send_text(self, text: str | None) -> None:
        """Send text chunk to ElevenLabs."""
        if not text or not text.strip():
            return
        
        ws = await self._ensure_connection()
        payload = {"text": text, "try_trigger_generation": False}
        await ws.send(json.dumps(payload))
    
    async def flush(self) -> None:
        """Signal end of text input."""
        ws = await self._ensure_connection()
        await ws.send(json.dumps({"text": ""}))
    
    async def receive_events(self) -> AsyncIterator[TTSEvent]:
        """Yield audio chunks as they arrive."""
        ws = await self._ensure_connection()
        
        async for raw_message in ws:
            message = json.loads(raw_message)
            
            if "audio" in message and message["audio"]:
                audio_chunk = base64.b64decode(message["audio"])
                if audio_chunk:
                    yield TTSEvent.chunk(audio_chunk)
            
            if message.get("isFinal"):
                break
    
    async def close(self) -> None:
        """Close WebSocket connection."""
        if self._ws:
            await self._ws.close()
            self._ws = None


async def tts_stream(
    event_stream: AsyncIterator,
) -> AsyncIterator:
    """
    Transform agent events to TTS audio events.
    
    Merges:
    - Upstream event passthrough
    - TTS audio chunk generation
    """
    tts = ElevenLabsTTS()
    
    async def process_upstream():
        """Process upstream and send text to TTS."""
        async for event in event_stream:
            yield event  # Pass through all events
            
            if event.type == "agent_chunk":
                await tts.send_text(event.text)
            elif event.type == "agent_end":
                await tts.flush()
    
    async def receive_audio():
        """Receive audio from TTS."""
        async for event in tts.receive_events():
            yield event
    
    try:
        # Merge both streams
        async for event in merge_async_iters(process_upstream(), receive_audio()):
            yield event
    finally:
        await tts.close()


async def merge_async_iters(*iterators):
    """Merge multiple async iterators."""
    queue = asyncio.Queue()
    done_count = 0
    total = len(iterators)
    
    async def consume(iterator):
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

## Cartesia Streaming TTS

Low-latency TTS alternative.

```python
import asyncio
import json
import os
from typing import AsyncIterator
import websockets

class CartesiaTTS:
    """Cartesia streaming TTS client."""
    
    def __init__(
        self,
        api_key: str | None = None,
        voice_id: str = "a0e99841-438c-4a64-b679-ae501e7d6091",
        model_id: str = "sonic-english",
        sample_rate: int = 16000,
    ):
        self.api_key = api_key or os.getenv("CARTESIA_API_KEY")
        self.voice_id = voice_id
        self.model_id = model_id
        self.sample_rate = sample_rate
        self._ws = None
    
    async def _ensure_connection(self):
        """Establish WebSocket connection."""
        if self._ws is None:
            url = f"wss://api.cartesia.ai/tts/websocket?api_key={self.api_key}"
            self._ws = await websockets.connect(url)
        return self._ws
    
    async def send_text(self, text: str, context_id: str = "default") -> None:
        """Send text for synthesis."""
        if not text.strip():
            return
        
        ws = await self._ensure_connection()
        message = {
            "model_id": self.model_id,
            "transcript": text,
            "voice": {"mode": "id", "id": self.voice_id},
            "context_id": context_id,
            "output_format": {
                "container": "raw",
                "encoding": "pcm_s16le",
                "sample_rate": self.sample_rate,
            },
            "continue": True,
        }
        await ws.send(json.dumps(message))
    
    async def receive_events(self) -> AsyncIterator[TTSEvent]:
        """Yield audio chunks."""
        ws = await self._ensure_connection()
        
        async for raw_message in ws:
            if isinstance(raw_message, bytes):
                yield TTSEvent.chunk(raw_message)
            else:
                message = json.loads(raw_message)
                if message.get("done"):
                    break
    
    async def close(self):
        """Close connection."""
        if self._ws:
            await self._ws.close()
            self._ws = None
```

## OpenAI TTS (Non-Streaming)

Simpler option for batch synthesis.

```python
import openai
import io

class OpenAITTS:
    """OpenAI TTS API client."""
    
    def __init__(
        self,
        api_key: str | None = None,
        voice: str = "alloy",
        model: str = "tts-1",
    ):
        self.client = openai.OpenAI(api_key=api_key)
        self.voice = voice
        self.model = model
    
    def synthesize(self, text: str) -> bytes:
        """Synthesize text to audio."""
        response = self.client.audio.speech.create(
            model=self.model,
            voice=self.voice,
            input=text,
            response_format="pcm",
        )
        return response.content
    
    async def synthesize_async(self, text: str) -> bytes:
        """Async wrapper for synthesis."""
        return await asyncio.to_thread(self.synthesize, text)


async def simple_tts_stream(
    event_stream: AsyncIterator,
) -> AsyncIterator:
    """Simple TTS that buffers text and synthesizes at agent_end."""
    tts = OpenAITTS()
    text_buffer = []
    
    async for event in event_stream:
        yield event
        
        if event.type == "agent_chunk":
            text_buffer.append(event.text)
        elif event.type == "agent_end" and text_buffer:
            full_text = "".join(text_buffer)
            audio = await tts.synthesize_async(full_text)
            yield TTSEvent.chunk(audio)
            text_buffer.clear()
```

## Voice Selection

### ElevenLabs Voice IDs

```python
ELEVENLABS_VOICES = {
    "rachel": "21m00Tcm4TlvDq8ikWAM",  # Female, neutral
    "drew": "29vD33N1CtxCmqQRPOHJ",    # Male, neutral
    "clyde": "2EiwWnXFnvU5JabPnv8n",   # Male, deep
    "dave": "CYw3kZ02Hs0563khs1Fj",    # Male, conversational
    "bella": "EXAVITQu4vr4xnSDxMaL",   # Female, soft
    "elli": "MF3mGyEYCl7XYWbV9V6O",    # Female, cheerful
    "sam": "yoZ06aMxZJJ28mfd3POQ",     # Male, narrative
}
```

### Cartesia Voice IDs

```python
CARTESIA_VOICES = {
    "british_lady": "79a125e8-cd45-4c13-8a67-188112f4dd22",
    "confident_woman": "a0e99841-438c-4a64-b679-ae501e7d6091",
    "friendly_man": "63ff761f-c1e8-414b-b969-d1833d1c870c",
    "professional_man": "694f9389-aac1-45b6-b726-9d9369183238",
}
```

## Audio Output Format

### PCM Specifications

Most TTS services output:
- Sample rate: 16000 Hz (configurable)
- Bit depth: 16-bit signed
- Channels: Mono

### Converting for Browser Playback

```python
import io
from pydub import AudioSegment

def pcm_to_wav(pcm_bytes: bytes, sample_rate: int = 16000) -> bytes:
    """Convert PCM to WAV for browser playback."""
    audio = AudioSegment(
        data=pcm_bytes,
        sample_width=2,
        frame_rate=sample_rate,
        channels=1,
    )
    buffer = io.BytesIO()
    audio.export(buffer, format="wav")
    return buffer.getvalue()

def pcm_to_mp3(pcm_bytes: bytes, sample_rate: int = 16000) -> bytes:
    """Convert PCM to MP3 for compression."""
    audio = AudioSegment(
        data=pcm_bytes,
        sample_width=2,
        frame_rate=sample_rate,
        channels=1,
    )
    buffer = io.BytesIO()
    audio.export(buffer, format="mp3", bitrate="64k")
    return buffer.getvalue()
```

## Error Handling

```python
import logging
from tenacity import retry, stop_after_attempt, wait_exponential

logger = logging.getLogger(__name__)

class TTSError(Exception):
    """TTS service error."""
    pass

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
async def connect_tts_with_retry(tts: ElevenLabsTTS):
    """Connect to TTS with retry."""
    try:
        await tts._ensure_connection()
    except Exception as e:
        logger.error(f"TTS connection failed: {e}")
        raise TTSError(f"Failed to connect to TTS: {e}")

async def safe_tts_stream(event_stream: AsyncIterator) -> AsyncIterator:
    """TTS stream with error handling."""
    tts = ElevenLabsTTS()
    
    try:
        await connect_tts_with_retry(tts)
        async for event in tts_stream(event_stream):
            yield event
    except TTSError as e:
        logger.error(f"TTS error: {e}")
        # Yield original events without audio
        async for event in event_stream:
            yield event
    finally:
        await tts.close()
```

## Latency Optimization

```python
class OptimizedTTS:
    """TTS with latency optimizations."""
    
    def __init__(self):
        self.tts = ElevenLabsTTS(
            model_id="eleven_turbo_v2_5",  # Fastest model
            output_format="pcm_16000",     # Raw PCM for speed
        )
        self._sentence_buffer = []
    
    async def send_with_early_trigger(self, text: str):
        """Send text with early synthesis trigger."""
        self._sentence_buffer.append(text)
        
        # Trigger synthesis on sentence boundaries
        full_text = "".join(self._sentence_buffer)
        if any(full_text.endswith(p) for p in ".!?"):
            await self.tts.send_text(full_text)
            await self.tts.send_text("")  # Trigger generation
            self._sentence_buffer.clear()
```
