# Frontend Patterns Reference

Browser-side audio capture and WebSocket communication for voice agents.

## Basic HTML/JavaScript Client

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Voice Agent</title>
    <style>
        body {
            font-family: system-ui, -apple-system, sans-serif;
            max-width: 600px;
            margin: 50px auto;
            padding: 20px;
        }
        .status {
            padding: 10px;
            border-radius: 8px;
            margin-bottom: 20px;
        }
        .status.connected { background: #d4edda; }
        .status.disconnected { background: #f8d7da; }
        .status.recording { background: #fff3cd; }
        
        button {
            padding: 15px 30px;
            font-size: 16px;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            margin: 5px;
        }
        button:disabled { opacity: 0.5; cursor: not-allowed; }
        
        #startBtn { background: #28a745; color: white; }
        #stopBtn { background: #dc3545; color: white; }
        
        .transcript {
            background: #f5f5f5;
            padding: 15px;
            border-radius: 8px;
            margin: 20px 0;
            min-height: 100px;
        }
        .transcript .partial { color: #666; font-style: italic; }
        .transcript .final { color: #000; }
    </style>
</head>
<body>
    <h1>🎤 Voice Agent</h1>
    
    <div id="status" class="status disconnected">Disconnected</div>
    
    <div>
        <button id="startBtn" onclick="startRecording()">Start Recording</button>
        <button id="stopBtn" onclick="stopRecording()" disabled>Stop Recording</button>
    </div>
    
    <div class="transcript">
        <h3>Transcript</h3>
        <div id="transcript"></div>
    </div>
    
    <div class="transcript">
        <h3>Agent Response</h3>
        <div id="response"></div>
    </div>

    <script>
        let ws = null;
        let mediaRecorder = null;
        let audioContext = null;
        let audioQueue = [];
        let isPlaying = false;
        
        const SAMPLE_RATE = 16000;
        const WS_URL = 'ws://localhost:8000/ws';
        
        // ========================================
        // WebSocket Connection
        // ========================================
        
        function connect() {
            ws = new WebSocket(WS_URL);
            
            ws.onopen = () => {
                updateStatus('connected', 'Connected');
                document.getElementById('startBtn').disabled = false;
            };
            
            ws.onclose = () => {
                updateStatus('disconnected', 'Disconnected');
                document.getElementById('startBtn').disabled = true;
                document.getElementById('stopBtn').disabled = true;
                // Reconnect after delay
                setTimeout(connect, 3000);
            };
            
            ws.onerror = (error) => {
                console.error('WebSocket error:', error);
            };
            
            ws.onmessage = async (event) => {
                if (event.data instanceof Blob) {
                    // Audio data
                    const audioData = await event.data.arrayBuffer();
                    queueAudio(audioData);
                } else {
                    // JSON event
                    const data = JSON.parse(event.data);
                    handleEvent(data);
                }
            };
        }
        
        function handleEvent(event) {
            switch (event.type) {
                case 'stt_chunk':
                    // Partial transcript
                    document.getElementById('transcript').innerHTML = 
                        `<span class="partial">${event.transcript}</span>`;
                    break;
                    
                case 'stt_output':
                    // Final transcript
                    document.getElementById('transcript').innerHTML = 
                        `<span class="final">${event.transcript}</span>`;
                    break;
                    
                case 'agent_chunk':
                    // Append agent text
                    document.getElementById('response').textContent += event.text;
                    break;
                    
                case 'agent_end':
                    // Agent finished
                    console.log('Agent turn complete');
                    break;
                    
                case 'tool_call':
                    console.log('Tool call:', event.tool, event.args);
                    break;
                    
                case 'tool_result':
                    console.log('Tool result:', event.tool, event.result);
                    break;
            }
        }
        
        // ========================================
        // Audio Recording
        // ========================================
        
        async function startRecording() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    audio: {
                        sampleRate: SAMPLE_RATE,
                        channelCount: 1,
                        echoCancellation: true,
                        noiseSuppression: true,
                    }
                });
                
                // Create audio context for processing
                audioContext = new AudioContext({ sampleRate: SAMPLE_RATE });
                const source = audioContext.createMediaStreamSource(stream);
                
                // Create script processor for raw PCM
                const processor = audioContext.createScriptProcessor(4096, 1, 1);
                
                processor.onaudioprocess = (e) => {
                    if (ws && ws.readyState === WebSocket.OPEN) {
                        const pcmData = convertToPCM16(e.inputBuffer.getChannelData(0));
                        ws.send(pcmData);
                    }
                };
                
                source.connect(processor);
                processor.connect(audioContext.destination);
                
                // Clear previous output
                document.getElementById('transcript').textContent = '';
                document.getElementById('response').textContent = '';
                
                updateStatus('recording', 'Recording...');
                document.getElementById('startBtn').disabled = true;
                document.getElementById('stopBtn').disabled = false;
                
            } catch (error) {
                console.error('Failed to start recording:', error);
                alert('Failed to access microphone. Please check permissions.');
            }
        }
        
        function stopRecording() {
            if (audioContext) {
                audioContext.close();
                audioContext = null;
            }
            
            updateStatus('connected', 'Connected');
            document.getElementById('startBtn').disabled = false;
            document.getElementById('stopBtn').disabled = true;
        }
        
        function convertToPCM16(float32Array) {
            const int16Array = new Int16Array(float32Array.length);
            for (let i = 0; i < float32Array.length; i++) {
                const s = Math.max(-1, Math.min(1, float32Array[i]));
                int16Array[i] = s < 0 ? s * 0x8000 : s * 0x7FFF;
            }
            return int16Array.buffer;
        }
        
        // ========================================
        // Audio Playback
        // ========================================
        
        async function queueAudio(arrayBuffer) {
            audioQueue.push(arrayBuffer);
            if (!isPlaying) {
                playNextAudio();
            }
        }
        
        async function playNextAudio() {
            if (audioQueue.length === 0) {
                isPlaying = false;
                return;
            }
            
            isPlaying = true;
            const data = audioQueue.shift();
            
            try {
                // Create audio context if needed
                if (!audioContext || audioContext.state === 'closed') {
                    audioContext = new AudioContext({ sampleRate: SAMPLE_RATE });
                }
                
                // Convert PCM to AudioBuffer
                const audioBuffer = pcmToAudioBuffer(data);
                
                // Play
                const source = audioContext.createBufferSource();
                source.buffer = audioBuffer;
                source.connect(audioContext.destination);
                source.onended = playNextAudio;
                source.start();
                
            } catch (error) {
                console.error('Audio playback error:', error);
                playNextAudio();
            }
        }
        
        function pcmToAudioBuffer(arrayBuffer) {
            const int16Array = new Int16Array(arrayBuffer);
            const float32Array = new Float32Array(int16Array.length);
            
            for (let i = 0; i < int16Array.length; i++) {
                float32Array[i] = int16Array[i] / 32768;
            }
            
            const audioBuffer = audioContext.createBuffer(1, float32Array.length, SAMPLE_RATE);
            audioBuffer.getChannelData(0).set(float32Array);
            
            return audioBuffer;
        }
        
        // ========================================
        // UI Helpers
        // ========================================
        
        function updateStatus(className, text) {
            const status = document.getElementById('status');
            status.className = `status ${className}`;
            status.textContent = text;
        }
        
        // Initialize
        connect();
    </script>
</body>
</html>
```

## React Component

```jsx
import React, { useState, useRef, useEffect, useCallback } from 'react';

const SAMPLE_RATE = 16000;

export function VoiceAgent({ wsUrl = 'ws://localhost:8000/ws' }) {
  const [status, setStatus] = useState('disconnected');
  const [transcript, setTranscript] = useState('');
  const [response, setResponse] = useState('');
  const [isRecording, setIsRecording] = useState(false);
  
  const wsRef = useRef(null);
  const audioContextRef = useRef(null);
  const audioQueueRef = useRef([]);
  const isPlayingRef = useRef(false);
  
  // WebSocket connection
  useEffect(() => {
    const connect = () => {
      const ws = new WebSocket(wsUrl);
      
      ws.onopen = () => setStatus('connected');
      ws.onclose = () => {
        setStatus('disconnected');
        setTimeout(connect, 3000);
      };
      
      ws.onmessage = async (event) => {
        if (event.data instanceof Blob) {
          const audioData = await event.data.arrayBuffer();
          queueAudio(audioData);
        } else {
          handleEvent(JSON.parse(event.data));
        }
      };
      
      wsRef.current = ws;
    };
    
    connect();
    
    return () => {
      if (wsRef.current) wsRef.current.close();
    };
  }, [wsUrl]);
  
  const handleEvent = useCallback((event) => {
    switch (event.type) {
      case 'stt_chunk':
        setTranscript(event.transcript);
        break;
      case 'stt_output':
        setTranscript(event.transcript);
        break;
      case 'agent_chunk':
        setResponse(prev => prev + event.text);
        break;
      case 'agent_end':
        break;
    }
  }, []);
  
  const startRecording = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({
        audio: { sampleRate: SAMPLE_RATE, channelCount: 1 }
      });
      
      const audioContext = new AudioContext({ sampleRate: SAMPLE_RATE });
      const source = audioContext.createMediaStreamSource(stream);
      const processor = audioContext.createScriptProcessor(4096, 1, 1);
      
      processor.onaudioprocess = (e) => {
        if (wsRef.current?.readyState === WebSocket.OPEN) {
          const pcm = convertToPCM16(e.inputBuffer.getChannelData(0));
          wsRef.current.send(pcm);
        }
      };
      
      source.connect(processor);
      processor.connect(audioContext.destination);
      
      audioContextRef.current = audioContext;
      setIsRecording(true);
      setTranscript('');
      setResponse('');
      setStatus('recording');
      
    } catch (error) {
      console.error('Recording error:', error);
    }
  };
  
  const stopRecording = () => {
    if (audioContextRef.current) {
      audioContextRef.current.close();
      audioContextRef.current = null;
    }
    setIsRecording(false);
    setStatus('connected');
  };
  
  const convertToPCM16 = (float32Array) => {
    const int16 = new Int16Array(float32Array.length);
    for (let i = 0; i < float32Array.length; i++) {
      const s = Math.max(-1, Math.min(1, float32Array[i]));
      int16[i] = s < 0 ? s * 0x8000 : s * 0x7FFF;
    }
    return int16.buffer;
  };
  
  const queueAudio = (arrayBuffer) => {
    audioQueueRef.current.push(arrayBuffer);
    if (!isPlayingRef.current) playNextAudio();
  };
  
  const playNextAudio = async () => {
    if (audioQueueRef.current.length === 0) {
      isPlayingRef.current = false;
      return;
    }
    
    isPlayingRef.current = true;
    const data = audioQueueRef.current.shift();
    
    const ctx = new AudioContext({ sampleRate: SAMPLE_RATE });
    const int16 = new Int16Array(data);
    const float32 = new Float32Array(int16.length);
    
    for (let i = 0; i < int16.length; i++) {
      float32[i] = int16[i] / 32768;
    }
    
    const buffer = ctx.createBuffer(1, float32.length, SAMPLE_RATE);
    buffer.getChannelData(0).set(float32);
    
    const source = ctx.createBufferSource();
    source.buffer = buffer;
    source.connect(ctx.destination);
    source.onended = () => {
      ctx.close();
      playNextAudio();
    };
    source.start();
  };
  
  return (
    <div className="voice-agent">
      <div className={`status ${status}`}>
        {status === 'connected' && '🟢 Connected'}
        {status === 'disconnected' && '🔴 Disconnected'}
        {status === 'recording' && '🔴 Recording...'}
      </div>
      
      <div className="controls">
        <button 
          onClick={startRecording}
          disabled={status !== 'connected'}
        >
          Start Recording
        </button>
        <button 
          onClick={stopRecording}
          disabled={!isRecording}
        >
          Stop Recording
        </button>
      </div>
      
      <div className="transcript">
        <h3>Transcript</h3>
        <p>{transcript}</p>
      </div>
      
      <div className="response">
        <h3>Agent Response</h3>
        <p>{response}</p>
      </div>
    </div>
  );
}
```

## Svelte Component

```svelte
<script>
  import { onMount, onDestroy } from 'svelte';
  
  export let wsUrl = 'ws://localhost:8000/ws';
  
  let status = 'disconnected';
  let transcript = '';
  let response = '';
  let isRecording = false;
  
  let ws;
  let audioContext;
  let audioQueue = [];
  let isPlaying = false;
  
  const SAMPLE_RATE = 16000;
  
  onMount(() => {
    connect();
  });
  
  onDestroy(() => {
    if (ws) ws.close();
    if (audioContext) audioContext.close();
  });
  
  function connect() {
    ws = new WebSocket(wsUrl);
    
    ws.onopen = () => status = 'connected';
    ws.onclose = () => {
      status = 'disconnected';
      setTimeout(connect, 3000);
    };
    
    ws.onmessage = async (event) => {
      if (event.data instanceof Blob) {
        const data = await event.data.arrayBuffer();
        queueAudio(data);
      } else {
        handleEvent(JSON.parse(event.data));
      }
    };
  }
  
  function handleEvent(event) {
    switch (event.type) {
      case 'stt_chunk':
      case 'stt_output':
        transcript = event.transcript;
        break;
      case 'agent_chunk':
        response += event.text;
        break;
    }
  }
  
  async function startRecording() {
    const stream = await navigator.mediaDevices.getUserMedia({
      audio: { sampleRate: SAMPLE_RATE, channelCount: 1 }
    });
    
    audioContext = new AudioContext({ sampleRate: SAMPLE_RATE });
    const source = audioContext.createMediaStreamSource(stream);
    const processor = audioContext.createScriptProcessor(4096, 1, 1);
    
    processor.onaudioprocess = (e) => {
      if (ws?.readyState === WebSocket.OPEN) {
        const pcm = convertToPCM16(e.inputBuffer.getChannelData(0));
        ws.send(pcm);
      }
    };
    
    source.connect(processor);
    processor.connect(audioContext.destination);
    
    isRecording = true;
    transcript = '';
    response = '';
    status = 'recording';
  }
  
  function stopRecording() {
    if (audioContext) {
      audioContext.close();
      audioContext = null;
    }
    isRecording = false;
    status = 'connected';
  }
  
  function convertToPCM16(float32Array) {
    const int16 = new Int16Array(float32Array.length);
    for (let i = 0; i < float32Array.length; i++) {
      const s = Math.max(-1, Math.min(1, float32Array[i]));
      int16[i] = s < 0 ? s * 0x8000 : s * 0x7FFF;
    }
    return int16.buffer;
  }
  
  function queueAudio(arrayBuffer) {
    audioQueue.push(arrayBuffer);
    if (!isPlaying) playNextAudio();
  }
  
  async function playNextAudio() {
    if (audioQueue.length === 0) {
      isPlaying = false;
      return;
    }
    
    isPlaying = true;
    const data = audioQueue.shift();
    
    const ctx = new AudioContext({ sampleRate: SAMPLE_RATE });
    const int16 = new Int16Array(data);
    const float32 = new Float32Array(int16.length);
    
    for (let i = 0; i < int16.length; i++) {
      float32[i] = int16[i] / 32768;
    }
    
    const buffer = ctx.createBuffer(1, float32.length, SAMPLE_RATE);
    buffer.getChannelData(0).set(float32);
    
    const source = ctx.createBufferSource();
    source.buffer = buffer;
    source.connect(ctx.destination);
    source.onended = () => {
      ctx.close();
      playNextAudio();
    };
    source.start();
  }
</script>

<div class="voice-agent">
  <div class="status {status}">
    {#if status === 'connected'}🟢 Connected{/if}
    {#if status === 'disconnected'}🔴 Disconnected{/if}
    {#if status === 'recording'}🔴 Recording...{/if}
  </div>
  
  <div class="controls">
    <button on:click={startRecording} disabled={status !== 'connected'}>
      Start Recording
    </button>
    <button on:click={stopRecording} disabled={!isRecording}>
      Stop Recording
    </button>
  </div>
  
  <div class="transcript">
    <h3>Transcript</h3>
    <p>{transcript}</p>
  </div>
  
  <div class="response">
    <h3>Agent Response</h3>
    <p>{response}</p>
  </div>
</div>

<style>
  .voice-agent {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
  }
  
  .status {
    padding: 10px;
    border-radius: 8px;
    margin-bottom: 20px;
  }
  
  .status.connected { background: #d4edda; }
  .status.disconnected { background: #f8d7da; }
  .status.recording { background: #fff3cd; }
  
  button {
    padding: 15px 30px;
    margin: 5px;
    border: none;
    border-radius: 8px;
    cursor: pointer;
  }
  
  button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>
```

## Audio Recording Best Practices

### Browser Permissions

```javascript
// Check microphone permission
async function checkMicrophonePermission() {
  try {
    const result = await navigator.permissions.query({ name: 'microphone' });
    return result.state; // 'granted', 'denied', 'prompt'
  } catch {
    return 'prompt'; // Assume prompt if API not supported
  }
}

// Request with user gesture
document.getElementById('startBtn').addEventListener('click', async () => {
  const permission = await checkMicrophonePermission();
  
  if (permission === 'denied') {
    alert('Microphone access denied. Please enable in browser settings.');
    return;
  }
  
  startRecording();
});
```

### Echo Cancellation

```javascript
// Recommended audio constraints
const audioConstraints = {
  audio: {
    sampleRate: 16000,
    channelCount: 1,
    echoCancellation: true,   // Prevent feedback
    noiseSuppression: true,   // Reduce background noise
    autoGainControl: true,    // Normalize volume
  }
};
```

### Mobile Support

```javascript
// Handle mobile audio context restrictions
async function initAudioContext() {
  const ctx = new (window.AudioContext || window.webkitAudioContext)({
    sampleRate: SAMPLE_RATE
  });
  
  // Resume on user interaction (required on mobile)
  if (ctx.state === 'suspended') {
    await ctx.resume();
  }
  
  return ctx;
}

// Add touch handler for iOS
document.addEventListener('touchstart', () => {
  if (audioContext?.state === 'suspended') {
    audioContext.resume();
  }
}, { once: true });
```
