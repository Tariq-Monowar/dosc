# Group Call — Frontend Developer Guide

One guide for integrating **group video/audio calls** in the Threads app. Covers Socket.IO events, payloads, push notifications, and MediaSoup flow.

**Conventions used in this doc**
- All IDs are **strings** (including `userId`, `conversationId`, `socketId`, `producerId`, `consumerId`, `transportId`) except `CallerInfo.id`, which is a number.
- `roomId` in Socket.IO is the **same value** as the conversation ID (use your conversation ID as `roomId` in `createRoom`).
- Callbacks are the last argument of `socket.emit(event, payload, callback)`. Always check for `res.error` before using the response.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Send vs Receive (what you send / what you get)](#2-send-vs-receive-what-you-send--what-you-get)
3. [Prerequisites](#3-prerequisites)
4. [Connection & First Steps](#4-connection--first-steps)
5. [Events You Emit](#5-events-you-emit)
6. [Events You Listen To](#6-events-you-listen-to)
7. [Push Notifications (FCM)](#7-push-notifications-fcm)
8. [Data Types (TypeScript-friendly)](#8-data-types-typescript-friendly)
9. [Call Flows](#9-call-flows)
10. [MediaSoup: Joining the Room](#10-mediasoup-joining-the-room)
11. [Errors](#11-errors)
12. [Quick Reference Table](#12-quick-reference-table)

---

## 1. Overview

- **Signalling:** Socket.IO (your backend).
- **Media:** MediaSoup (SFU). Each participant sends audio/video to the server; the server forwards it to others. No peer-to-peer.
- **Identity:** Server fills in names/avatars from the DB. You only send `userId` and `conversationId`; you never send `userInfo` for group calls.
- **Push:** When someone starts a call, offline (or background) members get an FCM data message with the same info as the socket event so you can show an incoming-call UI.

---

## 2. Send vs Receive (what you send / what you get)

Use this as the single place to see **what data you send** and **what data you get** for each event.

### Events where YOU send (emit) and get a response in the callback

| Event | **Data you SEND** | **Data you GET** (in callback) |
|-------|-------------------|----------------------------------|
| **join** | `userId` (string), e.g. `"42"` | — (no callback) |
| **get_active_calls** | `{}` or nothing | `{ calls: ActiveCall[] }` or `{ error: string }` |
| **group_call_initiate** | `{ callerId, conversationId, callType }` | — (no callback) |
| **createRoom** | `{ roomId: string }` (conversation ID) | `{ rtpCapabilities }` or `{ error: string }` |
| **createTransport** | `{ type: 'send' }` or `{ type: 'recv' }` | `{ id, iceParameters, iceCandidates, dtlsParameters }` or `{ error: string }` |
| **connectTransport** | `{ transportId, dtlsParameters }` | `{ success: true }` or `{ error: string }` |
| **produce** | `{ transportId, kind, rtpParameters }` | `{ id: string }` (producer id) or `{ error: string }` |
| **getProducers** | (no payload) | `{ producers: Producer[] }` or `{ error: string }` |
| **consume** | `{ transportId, producerId, rtpCapabilities }` | `{ id, producerId, kind, rtpParameters }` or `{ error: string }` |
| **resumeConsumer** | `{ consumerId }` | `{ success: true }` or `{ error: string }` |
| **media_state_change** | `{ video: boolean, audio: boolean }` | — (no callback) |
| **leaveRoom** | (no payload) | — (no callback) |

### Events where the SERVER sends and YOU only receive (you don’t emit these)

| Event | **Data you GET** (from server) |
|-------|--------------------------------|
| **active_group_calls** | `{ calls: Array<{ conversationId, conversationInfo, participantCount }> }` |
| **group_call_incoming** | `{ callerId, conversationId, callType, callerInfo, conversationInfo }` |
| **group_call_started** | `{ conversationId, conversationInfo }` (sometimes also `callerInfo`) |
| **group_call_ended** | `{ conversationId }` |
| **group_call_error** | `{ message: string }` |
| **newProducer** | `{ producerId, kind, socketId, participantInfo? }` |
| **media_state_change** | `{ video, audio, conversationId, socketId, userInfo }` |
| **participantLeft** | `{ socketId }` |
| **online-users** | `string[]` (list of online user IDs) |

### Object shapes you GET (from the tables above)

- **ActiveCall:** `{ conversationId: string, conversationInfo: ConversationInfo, participantCount: number }`
- **ConversationInfo:** `{ id: string, name: string, avatar: string \| null }`
- **CallerInfo:** `{ id: number, name: string, avatar: string \| null }`
- **ParticipantInfo / userInfo:** `{ userId: string, name: string, avatar: string \| null }`
- **Producer:** `{ id: string, kind: 'audio' \| 'video', socketId: string, participantInfo?: ParticipantInfo }`

---

## 3. Prerequisites

| Dependency | Purpose |
|------------|--------|
| `socket.io-client` (4.x) | Signalling |
| `mediasoup-client` (3.6.x) | WebRTC media (SFU) |

```bash
npm install socket.io-client mediasoup-client
```

- **Backend:** The Socket.IO server must be running and reachable at the URL you use in the client (same backend that serves your API).
- **Required before any group call step:** Connect to the server and emit `join` with the current user’s ID (see [Connection](#4-connection--first-steps)). No group-call or `get_active_calls` logic will work until after `join` has been emitted.

---

## 4. Connection & First Steps

### 4.1 Connect and register

```ts
import io from 'socket.io-client';

// Use your backend base URL (e.g. from env: process.env.API_URL or config)
const SOCKET_URL = 'https://your-api.com';  // no path suffix; Socket.IO uses /socket.io by default
const socket = io(SOCKET_URL, { path: '/socket.io', transports: ['websocket'] });

socket.on('connect', () => {
  socket.emit('join', userId);  // userId: string, e.g. "42"
});
```

- Replace `SOCKET_URL` with your actual backend URL (same origin or the API server that serves Socket.IO).
- You **must** call `socket.emit('join', userId)` after connect before using any group-call or `get_active_calls` logic.
- The server uses `userId` to route events and to know which conversations the user is in.

### 4.2 Active calls on app open

After `join`, the server may send **once** the list of group calls already running in the user’s conversations (no extra request needed):

```ts
socket.on('active_group_calls', (data: { calls: ActiveCall[] }) => {
  data.calls.forEach((call) => {
    // call.conversationId, call.conversationInfo, call.participantCount
    showJoinCallBanner(call);
  });
});
```

- If there are no active calls, this event is **not** emitted.
- For an explicit refresh (e.g. pull-to-refresh), use `get_active_calls` with a callback (see [Events You Emit](#5-events-you-emit)).

---

## 5. Events You Emit

All group-call-related events you send to the server. **Use the exact payload and callback shapes below; do not omit or rename fields.**

### 5.1 Table

| Event | Payload (exact) | Callback (if any) |
|-------|-----------------|-------------------|
| `join` | `userId` (string) | None |
| `get_active_calls` | `{}` or nothing | Optional. If provided: `(res) => {}` with `res.calls` or `res.error`. If not provided, server emits `active_group_calls` to you. |
| `group_call_initiate` | `{ callerId: string, conversationId: string, callType: 'video' \| 'audio' }` | None |
| `createRoom` | `{ roomId: string }` — **use the conversation ID as `roomId`** | Yes. See 4.2 below. |
| `createTransport` | `{ type: 'send' }` or `{ type: 'recv' }` | Yes. See 4.2 below. |
| `connectTransport` | `{ transportId: string, dtlsParameters: object }` — `dtlsParameters` from the transport’s `connect` event | Yes: `{ success: true }` or `{ error: string }` |
| `produce` | `{ transportId: string, kind: 'audio' \| 'video', rtpParameters: object }` — `kind` and `rtpParameters` from the send transport’s `produce` event | Yes: `{ id: string }` (producer id) or `{ error: string }` |
| `getProducers` | No payload. Use: `socket.emit('getProducers', (res) => { ... })` (callback as second argument) | Yes: `{ producers: Producer[] }` or `{ error: string }` |
| `consume` | `{ transportId: string, producerId: string, rtpCapabilities: object }` — `transportId` = your recv transport id, `rtpCapabilities` = `device.rtpCapabilities` | Yes. See 4.2 below. |
| `resumeConsumer` | `{ consumerId: string }` — the `id` you got from the `consume` callback | Yes: `{ success: true }` or `{ error: string }` |
| `media_state_change` | `{ video: boolean, audio: boolean }` — send both every time either changes | None |
| `leaveRoom` | No payload | None |

### 5.2 Exact callback response shapes

**createRoom**  
Callback receives one of:
- Success: `{ rtpCapabilities: RtpCapabilities }` (object to pass to `device.load({ routerRtpCapabilities })`)
- Error: `{ error: string }`

**createTransport**  
Callback receives one of:
- Success: `{ id: string, iceParameters: object, iceCandidates: object[], dtlsParameters: object }`  
  Pass this object (with the same keys) to `device.createSendTransport(...)` or `device.createRecvTransport(...)`.
- Error: `{ error: string }`

**consume**  
Callback receives one of:
- Success: `{ id: string, producerId: string, kind: 'audio' \| 'video', rtpParameters: object }`  
  Pass this to `recvTransport.consume({ id, producerId, kind, rtpParameters })`. Then call `resumeConsumer` with `consumerId: id`.
- Error: `{ error: string }`

### 5.3 Example: createRoom and getProducers

```ts
// createRoom — roomId is the conversation ID
socket.emit('createRoom', { roomId: conversationId }, (res: { rtpCapabilities?: RtpCapabilities; error?: string }) => {
  if (res.error) {
    console.error(res.error);
    return;
  }
  const { rtpCapabilities } = res;
  // next: device.load({ routerRtpCapabilities: rtpCapabilities })
});

// getProducers — callback is the second argument (no payload)
socket.emit('getProducers', (res: { producers?: Producer[]; error?: string }) => {
  if (res.error) return;
  const producers = res.producers ?? [];
  producers.filter((p) => p.socketId !== socket.id).forEach((p) => { /* consume p.id, p.kind */ });
});
```

---

## 6. Events You Listen To

All group-call-related events the server sends to the client. **Payload shapes are exact; use these types.**

| Event | When it fires | Payload shape (exact) |
|-------|----------------|------------------------|
| `active_group_calls` | After `join` (only if the user has ≥1 active call), or after `get_active_calls` when you did not pass a callback | `{ calls: ActiveCall[] }` — each item has `conversationId`, `conversationInfo`, `participantCount` |
| `group_call_incoming` | Someone started a call in a conversation you are in (you are not the caller) | `{ callerId: string, conversationId: string, callType: 'video' \| 'audio', callerInfo: CallerInfo, conversationInfo: ConversationInfo }` |
| `group_call_started` | First participant joined the MediaSoup room (call is live; others can join) | `{ conversationId: string, conversationInfo: ConversationInfo }` — sometimes also `callerInfo` when triggered from group_call_initiate |
| `group_call_ended` | Last participant left the room | `{ conversationId: string }` |
| `group_call_error` | You emitted `group_call_initiate` but it failed (e.g. not in group) | `{ message: string }` |
| `newProducer` | Another participant in the same room started sending a track | `{ producerId: string, kind: 'audio' \| 'video', socketId: string, participantInfo?: ParticipantInfo }` — call `consume` then `resumeConsumer` for this `producerId`; do not consume if `socketId === socket.id` |
| `media_state_change` | Another participant toggled camera or mic | `{ video: boolean, audio: boolean, conversationId: string, socketId: string, userInfo: ParticipantInfo \| null }` |
| `participantLeft` | A participant left the room | `{ socketId: string }` — remove or hide the UI for that participant |
| `online-users` | Broadcast when any user connects or disconnects | `string[]` — list of online user IDs |

**What to do with each**

- **Incoming call screen:** Use `group_call_incoming` (or the same data from FCM when in background). Use `callerInfo` and `conversationInfo` for name/avatar; use `conversationId` to join when user taps Accept.
- **“Call in progress” banner:** Show when you have `group_call_started` or an entry in `active_group_calls` for that conversation. Hide when you receive `group_call_ended` for that `conversationId`.
- **Remote video/audio:** For each `newProducer` (and each producer from `getProducers` when you join mid-call), call `consume` then `resumeConsumer`, then attach `consumer.track` to a `<video>` or `<audio>` element. Map `socketId` (and optional `participantInfo`) to the tile so you can show name/avatar and remove the tile on `participantLeft`.
- **Camera/mic indicators:** Use `media_state_change`: `socketId` identifies the participant, `video`/`audio` are the new state, `userInfo` for name/avatar.

---

## 7. Push Notifications (FCM)

When someone starts a group call, the server sends a **Firebase Cloud Messaging (FCM) data message** to every other member of the conversation (including when the app is in background or killed). Use it to show an incoming-call UI with the same info as `group_call_incoming`.

### 7.1 When the server sends it

- On `group_call_initiate`: for each member **except the caller**, the server sends one FCM message (to all of that user’s FCM tokens). So as a frontend you may receive one or more messages per call invite.

### 7.2 Where to read the payload

- In your FCM handler you receive a **remote message** object. The **data** part is a flat map: all keys and values are strings.
- Check `message.data.type === 'group_call_initiate'` to treat it as a group call.
- Use `message.data.conversationId` directly (string).
- The key `message.data.data` is a **JSON string**. Parse it once: `const payload = JSON.parse(message.data.data)`.

### 7.3 Data keys (all strings in `message.data`)

| Key in `message.data` | Meaning |
|------------------------|--------|
| `type` | Always `"group_call_initiate"` for group calls |
| `title` | Conversation name (for notification title if you show one) |
| `body` | e.g. "Alice is calling" (for notification body) |
| `message` | e.g. "Alice started a group call" |
| `conversationId` | Conversation ID — **use this to join the call** when user taps Accept |
| `data` | JSON string — parse with `JSON.parse(message.data.data)` |

### 7.4 Shape of parsed `data` (after JSON.parse(message.data.data))

```ts
{
  callerId: string;
  callType: 'video' | 'audio';
  conversationId: string;
  callerInfo: { id: number; name: string; avatar: string | null };
  conversationInfo: { id: string; name: string; avatar: string | null };
}
```

Use `callerInfo` and `conversationInfo` for the incoming-call screen (name, avatar). Use `conversationId` when the user taps Accept to run your join-room flow.

### 7.5 What to do in the app

- **App in foreground:** Prefer the socket event `group_call_incoming` (same data shape). You can ignore or dedupe the FCM message if you already showed the UI from the socket.
- **App in background or killed:** Handle only the FCM data message: if `type === 'group_call_initiate'`, parse `data`, show incoming-call UI. On Accept: open app, connect Socket.IO, call `join(userId)`, then run the [MediaSoup join flow](#10-mediasoup-joining-the-room) with `conversationId` from the payload.

Push is only for **notifying**; joining and media use Socket.IO + MediaSoup.

---

## 8. Data Types (TypeScript-friendly)

Use these for typing payloads and handlers.

```ts
// Active call (from active_group_calls / get_active_calls)
interface ActiveCall {
  conversationId: string;
  conversationInfo: ConversationInfo;
  participantCount: number;
}

interface ConversationInfo {
  id: string;
  name: string;
  avatar: string | null;
}

// Caller (incoming call / push)
interface CallerInfo {
  id: number;
  name: string;
  avatar: string | null;
}

// Participant in room (newProducer, getProducers, media_state_change)
interface ParticipantInfo {
  userId: string;
  name: string;
  avatar: string | null;
}

interface Producer {
  id: string;
  kind: 'audio' | 'video';
  socketId: string;
  participantInfo?: ParticipantInfo;
}

// media_state_change (received)
interface MediaStateChangePayload {
  video: boolean;
  audio: boolean;
  conversationId: string;
  socketId: string;
  userInfo: ParticipantInfo | null;
}
```

---

## 9. Call Flows

“Join room” below means: run the full [MediaSoup: Joining the Room](#10-mediasoup-joining-the-room) sequence (createRoom → load device → create transports → produce → getProducers → consume each → handle newProducer) using the given `conversationId` as `roomId`.

### 9.1 You start a call

1. Emit: `socket.emit('group_call_initiate', { callerId: currentUserId, conversationId, callType: 'video' | 'audio' })`.
2. Immediately run the **join room** flow (Section 10) with this same `conversationId`.
3. Other members get `group_call_incoming` (socket) and/or FCM with `type: 'group_call_initiate'`. When the first participant (you) enters the room, everyone gets `group_call_started`.

### 9.2 Someone else starts a call (you receive)

- **App in foreground:** On `group_call_incoming`, show incoming-call UI using `callerInfo` and `conversationInfo`. When the user taps **Accept**, run the **join room** flow (Section 10) with `data.conversationId`.
- **App in background or killed:** On FCM with `type === 'group_call_initiate'`, parse `message.data.data` and show the same UI. When the user taps **Accept**, open the app, connect the socket, emit `join(userId)`, then run the **join room** flow with `conversationId` from the parsed payload.

### 9.3 You join a call already in progress

- You already have a `conversationId` from `group_call_started` or from `active_group_calls` (e.g. user tapped a “Join call” banner). When the user taps **Join**, run the **join room** flow (Section 10) with that `conversationId`. The flow includes `getProducers` and consuming each existing producer.

### 9.4 You leave the call

- Run your leave cleanup (stop tracks, close producers/consumers/transports), then emit `socket.emit('leaveRoom')`. Other participants receive `participantLeft` with your `socketId`. If you were the last in the room, everyone (including you) receives `group_call_ended` with that `conversationId`.

---

## 10. MediaSoup: Joining the Room

Follow this order exactly. You must have: socket connected, `join(userId)` already emitted, and the `conversationId` for the call.

**Step 1 — createRoom**  
- Emit: `socket.emit('createRoom', { roomId: conversationId }, callback)`.  
- In the callback: if `res.error`, stop. Otherwise set `const rtpCapabilities = res.rtpCapabilities`.

**Step 2 — Load the device**  
- `const device = new mediasoupClient.Device()`  
- `await device.load({ routerRtpCapabilities: rtpCapabilities })`  
- Do not create transports before the device has loaded.

**Step 3 — Create send transport**  
- Emit: `socket.emit('createTransport', { type: 'send' }, callback)`.  
- In the callback: if `res.error`, stop. Otherwise create the transport:  
  `sendTransport = device.createSendTransport({ id: res.id, iceParameters: res.iceParameters, iceCandidates: res.iceCandidates, dtlsParameters: res.dtlsParameters })`.  
- On `sendTransport.on('connect', ({ dtlsParameters }, callback, errback)`:  
  Emit `socket.emit('connectTransport', { transportId: sendTransport.id, dtlsParameters }, (res) => { if (res.error) errback(new Error(res.error)); else callback(); })`.  
- On `sendTransport.on('produce', ({ kind, rtpParameters }, callback, errback)`:  
  Emit `socket.emit('produce', { transportId: sendTransport.id, kind, rtpParameters }, (res) => { if (res.error) errback(new Error(res.error)); else callback({ id: res.id }); })`.

**Step 4 — Create recv transport**  
- Emit: `socket.emit('createTransport', { type: 'recv' }, callback)`.  
- In the callback: if `res.error`, stop. Otherwise:  
  `recvTransport = device.createRecvTransport({ id: res.id, iceParameters: res.iceParameters, iceCandidates: res.iceCandidates, dtlsParameters: res.dtlsParameters })`.  
- On `recvTransport.on('connect', ({ dtlsParameters }, callback, errback)`:  
  Emit `socket.emit('connectTransport', { transportId: recvTransport.id, dtlsParameters }, (res) => { if (res.error) errback(new Error(res.error)); else callback(); })`.

**Step 5 — Get user media and produce**  
- Get stream: `const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true })` (or your constraints).  
- For the audio track: `sendTransport.produce({ track: stream.getAudioTracks()[0] })` — this triggers the transport’s `produce` handler above and sends `produce` to the server.  
- For the video track: `sendTransport.produce({ track: stream.getVideoTracks()[0] })`.  
- The server will emit `newProducer` to other participants; you do not need to consume your own.

**Step 6 — Get existing producers and consume**  
- Emit: `socket.emit('getProducers', (res) => { ... })`.  
- If `res.error`, skip. Otherwise for each `p` in `res.producers`: if `p.socketId === socket.id`, skip.  
- For each other producer:  
  - Emit `socket.emit('consume', { transportId: recvTransport.id, producerId: p.id, rtpCapabilities: device.rtpCapabilities }, (res) => { ... })`.  
  - If `res.error`, log and skip.  
  - Create consumer: `const consumer = await recvTransport.consume({ id: res.id, producerId: res.producerId, kind: res.kind, rtpParameters: res.rtpParameters })`.  
  - Emit `socket.emit('resumeConsumer', { consumerId: consumer.id }, () => {})`.  
  - Attach to playback: e.g. `videoElement.srcObject = new MediaStream([consumer.track]); videoElement.play()` (use a `<video>` for kind `video`, `<audio>` or same for `audio`).  
  - Store a mapping from `p.socketId` (or producer id) to the consumer/video element so you can remove it on `participantLeft`.

**Step 7 — Handle new producers (someone joined later)**  
- Listen: `socket.on('newProducer', async ({ producerId, kind, socketId, participantInfo }) => { ... })`.  
- If `socketId === socket.id`, return.  
- Same as step 6: emit `consume` with this `producerId`, then `recvTransport.consume(...)`, then `resumeConsumer`, then attach the track to a new video/audio element. Use `socketId` and `participantInfo` for the tile label.

**Step 8 — Leave**  
- Stop your local stream: `stream.getTracks().forEach((t) => t.stop())`.  
- Close all producers (from send transport).  
- Close all consumers and the recv transport.  
- Close the send transport.  
- Emit: `socket.emit('leaveRoom')`.  
- Others will receive `participantLeft` with your `socketId`; you may receive `group_call_ended` if you were the last.

**Important:** `roomId` in `createRoom` is the **conversation ID**. Both send and recv transports must complete the `connectTransport` step (the server does the DTLS handshake for both).

---

## 11. Errors

Every emit that takes a callback can receive `{ error: string }` instead of success. Always check `res.error` and handle it (e.g. show a toast or retry); do not assume success.

**Errors by event (exact strings from server)**

| Event | Possible `error` or `message` values |
|-------|--------------------------------------|
| **group_call_error** (socket event) | `"You are not in this group"`, `"Invalid caller id"` |
| **createRoom** (callback) | `"roomId required"`, `"Join first with your user ID"`, `"You are not in this group"` |
| **createTransport** (callback) | `"Participant not found"` |
| **connectTransport** (callback) | `"Participant not found"`, `"Transport not found"` |
| **produce** (callback) | `"Participant not found"`, `"Transport not found"` |
| **getProducers** (callback) | `"Participant not found"` |
| **consume** (callback) | `"Participant not found"`, `"Producer not found"`, `"Cannot consume own producer"`, `"RTP capabilities mismatch"`, `"Receive transport not found"` |
| **resumeConsumer** (callback) | `"Participant not found"`, `"Consumer not found"` |
| **get_active_calls** (callback) | `"Not joined"`, `"Invalid user id"` |

**media_state_change:** If the user is not in a room, the server ignores the emit and does not send any error back.

**Suggested handling:** For “Join first with your user ID”, ensure you have called `join(userId)` after connect. For “You are not in this group”, do not let the user start or join that conversation’s call. For “RTP capabilities mismatch”, ensure the device is loaded and you are passing `device.rtpCapabilities` to `consume`.

---

## 12. Quick Reference Table

**Emit (client → server)** — use these exact payloads.

| Event | Payload |
|-------|--------|
| `join` | `userId` (string) — single argument, no object |
| `get_active_calls` | `{}` or no payload; callback as second arg: `(res) => {}` |
| `group_call_initiate` | `{ callerId: string, conversationId: string, callType: 'video' \| 'audio' }` |
| `createRoom` | `{ roomId: string }` — `roomId` = conversation ID |
| `createTransport` | `{ type: 'send' }` or `{ type: 'recv' }` |
| `connectTransport` | `{ transportId: string, dtlsParameters: object }` |
| `produce` | `{ transportId: string, kind: 'audio' \| 'video', rtpParameters: object }` |
| `getProducers` | No payload; callback only: `socket.emit('getProducers', (res) => {})` |
| `consume` | `{ transportId: string, producerId: string, rtpCapabilities: object }` |
| `resumeConsumer` | `{ consumerId: string }` |
| `media_state_change` | `{ video: boolean, audio: boolean }` |
| `leaveRoom` | No payload |

**Listen (server → client)** — payload shapes.

| Event | Payload |
|-------|--------|
| `active_group_calls` | `{ calls: Array<{ conversationId, conversationInfo, participantCount }> }` |
| `group_call_incoming` | `{ callerId, conversationId, callType, callerInfo, conversationInfo }` |
| `group_call_started` | `{ conversationId, conversationInfo }` (+ optional `callerInfo`) |
| `group_call_ended` | `{ conversationId }` |
| `group_call_error` | `{ message: string }` |
| `newProducer` | `{ producerId, kind, socketId, participantInfo? }` |
| `media_state_change` | `{ video, audio, conversationId, socketId, userInfo }` |
| `participantLeft` | `{ socketId }` |
| `online-users` | `string[]` |

---

**End of guide.** This document is self-contained; a frontend developer can implement the full group call flow without other references.
