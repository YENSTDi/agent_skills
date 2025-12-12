# STT Patterns Reference

Speech-to-Text integration patterns for voice agents.

## AssemblyAI Real-time STT

Production-ready streaming transcription with turn detection.

```python
import asyncio
import json
import os
from typing import AsyncIterator
import websockets
from websockets.client import WebSocketClientProtocol

class STTEvent:
    """Base STT event."""
    def __init__(self, type: str, transcript: str = ""):
        self.type = type
        self.transcript = transcript
    
    @classmethod
    def chunk(cls, text: str) -> "STTEvent":
        return cls("stt_chunk", text)
    
    @classmethod
    def output(cls, text: str) -> "STTEvent":
        return cls("stt_output", text)


class AssemblyAISTT:
    """AssemblyAI real-time streaming client."""
    
    def __init__(
        self,
        api_key: str | None = None,
        sample_rate: int = 16000,
    ):
        self.api_key = api_key or os.getenv("ASSEMBLYAI_API_KEY")
        self.sample_rate = sample_rate
        self._ws: WebSocketClientProtocol | None = None
    
    async def _ensure_connection(self) -> WebSocketClientProtocol:
        """Establish WebSocket connection to AssemblyAI."""
        if self._ws is None:
            url = (
                f"wss://streaming.assemblyai.com/v3/ws"
                f"?sample_rate={self.sample_rate}&format_turns=true"
            )
            self._ws = await websockets.connect(
                url,
                additional_headers={"Authorization": self.api_key},
            )
        return self._ws
    
    async def send_audio(self, audio_chunk: bytes) -> None:
        """Send PCM audio bytes to AssemblyAI."""
        ws = await self._ensure_connection()
        await ws.send(audio_chunk)
    
    async def receive_events(self) -> AsyncIterator[STTEvent]:
        """Yield STT events as they arrive."""
        ws = await self._ensure_connection()
        async for raw_message in ws:
            message = json.loads(raw_message)
            
            if message.get("type") == "Turn":
                transcript = message.get("transcript", "")
                if message.get("turn_is_formatted"):
                    # Final formatted transcript
                    yield STTEvent.output(transcript)
                else:
                    # Partial transcript
                    yield STTEvent.chunk(transcript)
    
    async def close(self) -> None:
        """Close the WebSocket connection."""
        if self._ws:
            await self._ws.close()
            self._ws = None


async def stt_stream(
    audio_stream: AsyncIterator[bytes],
    sample_rate: int = 16000,
) -> AsyncIterator[STTEvent]:
    """
    Transform audio stream to STT events.
    
    Uses producer-consumer pattern:
    - Producer: sends audio chunks to AssemblyAI
    - Consumer: receives transcription events
    """
    stt = AssemblyAISTT(sample_rate=sample_rate)
    
    async def send_audio():
        """Background task to pump audio to STT."""
        try:
            async for chunk in audio_stream:
                await stt.send_audio(chunk)
        finally:
            await stt.close()
    
    send_task = asyncio.create_task(send_audio())
    
    try:
        async for event in stt.receive_events():
            yield event
    finally:
        send_task.cancel()
        try:
            await send_task
        except asyncio.CancelledError:
            pass
        await stt.close()
```

## OpenAI Whisper (Local/API)

For non-streaming or batch transcription.

```python
import openai
import tempfile
from pathlib import Path

class WhisperSTT:
    """OpenAI Whisper API client."""
    
    def __init__(self, api_key: str | None = None):
        self.client = openai.OpenAI(api_key=api_key)
    
    async def transcribe(self, audio_bytes: bytes, format: str = "wav") -> str:
        """Transcribe audio bytes to text."""
        with tempfile.NamedTemporaryFile(suffix=f".{format}", delete=False) as f:
            f.write(audio_bytes)
            f.flush()
            
            with open(f.name, "rb") as audio_file:
                response = self.client.audio.transcriptions.create(
                    model="whisper-1",
                    file=audio_file,
                    response_format="text",
                )
            
            Path(f.name).unlink()
        
        return response


# Local Whisper with faster-whisper
from faster_whisper import WhisperModel

class LocalWhisperSTT:
    """Local Whisper using faster-whisper."""
    
    def __init__(self, model_size: str = "base"):
        self.model = WhisperModel(model_size, device="cpu", compute_type="int8")
    
    def transcribe(self, audio_path: str) -> str:
        """Transcribe audio file to text."""
        segments, _ = self.model.transcribe(audio_path)
        return " ".join(segment.text for segment in segments)
```

## Deepgram Streaming STT

```python
import json
from deepgram import (
    DeepgramClient,
    LiveTranscriptionEvents,
    LiveOptions,
)

class DeepgramSTT:
    """Deepgram streaming client."""
    
    def __init__(self, api_key: str | None = None):
        self.client = DeepgramClient(api_key)
        self._connection = None
        self._transcript_queue = asyncio.Queue()
    
    async def connect(self, sample_rate: int = 16000):
        """Establish streaming connection."""
        options = LiveOptions(
            model="nova-2",
            punctuate=True,
            language="en-US",
            encoding="linear16",
            sample_rate=sample_rate,
            channels=1,
            interim_results=True,
            utterance_end_ms=1000,
            vad_events=True,
        )
        
        self._connection = self.client.listen.live.v("1")
        
        @self._connection.on(LiveTranscriptionEvents.Transcript)
        async def on_transcript(_, result):
            transcript = result.channel.alternatives[0].transcript
            is_final = result.is_final
            if transcript:
                await self._transcript_queue.put(
                    STTEvent.output(transcript) if is_final 
                    else STTEvent.chunk(transcript)
                )
        
        await self._connection.start(options)
    
    async def send_audio(self, audio_chunk: bytes):
        """Send audio to Deepgram."""
        if self._connection:
            await self._connection.send(audio_chunk)
    
    async def receive_events(self) -> AsyncIterator[STTEvent]:
        """Yield transcription events."""
        while True:
            event = await self._transcript_queue.get()
            yield event
    
    async def close(self):
        """Close connection."""
        if self._connection:
            await self._connection.finish()
```

## Audio Format Considerations

### PCM Audio Specs

Most STT services expect:
- Sample rate: 16000 Hz
- Bit depth: 16-bit
- Channels: Mono (1 channel)
- Format: Raw PCM (no headers)

### Converting Audio Formats

```python
from pydub import AudioSegment
import io

def convert_to_pcm(audio_bytes: bytes, input_format: str = "webm") -> bytes:
    """Convert audio to PCM format for STT."""
    audio = AudioSegment.from_file(io.BytesIO(audio_bytes), format=input_format)
    audio = audio.set_frame_rate(16000).set_channels(1).set_sample_width(2)
    return audio.raw_data

def convert_to_wav(pcm_bytes: bytes, sample_rate: int = 16000) -> bytes:
    """Convert PCM to WAV for APIs that need headers."""
    audio = AudioSegment(
        data=pcm_bytes,
        sample_width=2,
        frame_rate=sample_rate,
        channels=1,
    )
    buffer = io.BytesIO()
    audio.export(buffer, format="wav")
    return buffer.getvalue()
```

## Error Handling

```python
import logging
from tenacity import retry, stop_after_attempt, wait_exponential

logger = logging.getLogger(__name__)

class STTError(Exception):
    """STT service error."""
    pass

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
async def connect_stt_with_retry(stt: AssemblyAISTT):
    """Connect to STT with automatic retry."""
    try:
        await stt._ensure_connection()
    except Exception as e:
        logger.error(f"STT connection failed: {e}")
        raise STTError(f"Failed to connect to STT: {e}")

async def safe_stt_stream(
    audio_stream: AsyncIterator[bytes],
) -> AsyncIterator[STTEvent]:
    """STT stream with error handling."""
    stt = AssemblyAISTT()
    
    try:
        await connect_stt_with_retry(stt)
        async for event in stt.receive_events():
            yield event
    except STTError as e:
        logger.error(f"STT error: {e}")
        yield STTEvent.output("[Transcription unavailable]")
    finally:
        await stt.close()
```
