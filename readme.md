# Eniac S2S — Real‑Time Speech‑to‑Speech API

![status](https://img.shields.io/badge/status-beta-yellow)

Eniac S2S is a low‑latency, **speech‑to‑speech (S2S)** service that streams audio in, routes it through your choice of **STT → LLM → TTS** stack, and streams natural speech back — all over a single WebSocket connection.

It handles the messy parts of building voice agents — **latency, interruption, turn‑taking, session orchestration, and scaling infrastructure** — so you can focus on delightful conversational experiences.

---

## Why we built Eniac S2S

1. **Model flexibility without glue‑code** — pick any TTS, STT, or LLM per request to match language, latency, or cost.
2. **OpenAI‑Realtime feel, but cheaper & more customisable** — same streaming UX with a frugal, fully‑pluggable stack.

---

## Base URL & Auth

| Item                   | Value                                                                  |
| ---------------------- | ---------------------------------------------------------------------- |
| **WebSocket endpoint** | `wss://s2s.geteniac.tech/ws`                                           |
| **Secret key**         | [Request access »](https://forms.gle/bmUt4gyZ1n4jRjPY8) |
| **Health check**       | `GET /api/health` → `{ "status": "healthy" }`                          |

Include your key in the first `session.start` message (see below). A key is required for **every** connection.

---

## Provider Enums

| Field          | Allowed values                            | Default      |
| -------------- | ----------------------------------------- | ------------ |
| `tts_provider` | `elevenlabs`, `sarvam`, `cartesia`        | `elevenlabs` |
| `llm_provider` | `openai`, `anthropic`, `groq`, `cerebras` | `openai`     |
| `stt_provider` | `deepgram`, `assemblyai`                  | `deepgram`   |

> Need another provider? Open a PR or drop us a line.

---

```ts
import WebSocket from "ws";

const ws = new WebSocket("wss://s2s.geteniac.tech/ws");

ws.on("open", () => {
  /* ① kick‑off the session */
  ws.send(JSON.stringify({
    type: "session.start",
    instruction: "You are a helpful assistant.",
    initial_prompt: "Introduce yourself as Jane, an AI assistant specialised in customer support.",
    tts_provider: "elevenlabs",
    stt_provider: "deepgram",
    llm_provider: "openai",
    secret: process.env.ENIAC_SECRET
  }));

  /* ② stream raw 16‑kHz mono PCM frames (base64) */
  microphone.on("frame", (chunk) => {
    ws.send(JSON.stringify({
      type: "input",
      audio: chunk.toString("base64")
    }));
  });
});

ws.on("message", (raw) => {
  const e = JSON.parse(raw.toString());
  // handle events below …
});
```

---

## Event Reference

### Envelope

Every payload is a JSON object with at least a `type` field (and sometimes a `service`).

```json
{
  "type": "…",               // event name
  /* service‑specific fields */
}
```

> **Direction legend**  ⬅️ server → client | ➡️ client → server

---

### INPUT (➡️)

| Event                  | Purpose                              |
| ---------------------- | ------------------------------------ |
| `session.start`        | Start a session & choose providers   |
| `input`                | Stream a base64‑encoded audio chunk  |
| `function_call_result` | Return the result of a tool/function |

#### `session.start`

```json
{
  "type": "session.start",
  "instruction": "You are a helpful assistant that can answer questions about the weather.",
  "initial_message": "Introduce yourself as Jane, an AI assistant specialized in customer support.",
  "tts_provider": "elevenlabs",
  "stt_provider": "deepgram",
  "llm_provider": "openai",
  "secret": "<YOUR‑SECRET>",
  "tools": []
}
```

#### `input` — Audio Chunk

```json
{
  "type": "input",
  "audio": "<base64‑pcm>"
}
```

#### `function_call_result`

```json
{
  "type": "function_call_result",
  "function_name": "get_temperature",
  "tool_call_id": "call_123",
  "arguments": { "location": "New York" },
  "result": "72°F"
}
```

---

### OUTPUT (⬅️)

| Event family      | Events                                                                          | Description                        |              |                           |
| ----------------- | ------------------------------------------------------------------------------- | ---------------------------------- | ------------ | ------------------------- |
| **Interruption**  | `interruption`, `start_interruption`, `stop_interruption`, `status:interrupted` | Cooperative barge‑in / turn‑taking |              |                           |
| **Transcription** | `transcription`                                                                 | Interim or final STT text          |              |                           |
| **TTS**           | `service:"tts"` → `speak (start)`, `response (audio)`                           | TTS engine progress & audio frames |              |                           |
| **LLM**           | `service:"llm"` → `llm_text`, `response(start)`, `response(end)`                | Token stream & lifecycle           |              |                           |
| **Function Call** | \`function\_call (in\_progress                                                  | result                             | cancelled)\` | Tool invocation lifecycle |

#### Interruption

```json
{ "type": "start_interruption" }
{ "type": "stop_interruption" }
```

#### Transcription

```json
// interim
{
  "type": "transcription",
  "message": "Yeah. I want to know if you can help me with the",
  "interim": true,
  "event_tag": "interim_transcription"
}

// final
{
  "type": "transcription",
  "message": "Yeah. I want to know if you can help me with the",
  "interim": false,
  "event_tag": "final_transcription"
}
```

> **Final vs. interim** — a transcription is *final* when it is followed by an LLM event, an interruption, or a long silence.

#### TTS

```json
{
  "service": "tts",
  "type": "speak",
  "text": "The temperature is 72 °F.",
  "update": "start"
}
{
  "type": "response",
  "audio": "<base64‑pcm>"
}
```

#### Function Call Lifecycle

```json
// request
{
  "service": "llm",
  "type": "function_call",
  "update": "in_progress",
  "function_name": "get_temperature",
  "tool_call_id": "call_123",
  "arguments": "{\"location\":\"New York\"}"
}

// result sent by client
{
  "type": "function_call_result",
  "function_name": "get_temperature",
  "tool_call_id": "call_123",
  "arguments": { "location": "New York" },
  "result": "72°F"
}
```

---

## Example Tool Flow

```json
// ① LLM triggers a function
{
  "service": "llm",
  "type": "function_call",
  "function_name": "get_temperature",
  "arguments": "{\"location\":\"New York\"}"
}

// ② Client replies to the server
{
  "type": "function_call_result",
  "function_name": "get_temperature",
  "arguments": { "location": "New York" },
  "result": "72°F"
}
```

---

## About

A project by **[GetEniac](https://geteniac.tech)** — we build phone voice agents for conversational AI.
