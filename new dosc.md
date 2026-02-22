# Group Call — Socket Integration Guide

> For frontend developers integrating real-time group video/audio calls into the **Threads** mobile or web app.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Dependencies](#dependencies)
4. [Connection Setup](#connection-setup)
5. [Socket Events Reference](#socket-events-reference)
   - [Events You Emit (Client → Server)](#events-you-emit-client--server)
   - [Events You Listen To (Server → Client)](#events-you-listen-to-server--client)
6. [Full Call Flow — Step by Step](#full-call-flow--step-by-step)
   - [Caller Side](#caller-side)
   - [Receiver Side](#receiver-side)
   - [Joining a Call Already in Progress](#joining-a-call-already-in-progress)
7. [MediaSoup WebRTC Setup](#mediasoup-webrtc-setup)
8. [Participant Info & Conversation Info](#participant-info--conversation-info)
9. [Data Shapes](#data-shapes)
10. [Error Handling](#error-handling)
11. [Edge Cases](#edge-cases)

---

## Overview

Group calls in Threads use **Socket.IO** for signalling and **MediaSoup** (SFU) for media routing. Unlike peer-to-peer calls, every participant sends their media to the server, and the server distributes it to everyone else. This means:

- No direct peer connections between users.
- The server handles all media routing.
- Participants can join or leave at any time without breaking others' connections.
- User name, avatar, and conversation info are all sent by the server automatically — you do **not** need to pass them manually from the frontend.

---

## Architecture

```
Flutter / Web App
       │
       │  Socket.IO (signalling)
       ▼
  Threads Backend  ──── MediaSoup Worker (SFU)
       │                      │
       │  group_call_incoming  │  Audio/Video streams
       ▼                      ▼
  Other Participants  ◄──── MediaSoup Router (per conversation)
```

---

## Dependencies

| Library | Version | Purpose |
|---|---|---|
| `socket.io-client` | 4.x | Signalling |
| `mediasoup-client` | 3.6.x | WebRTC media (SFU) |

**Install:**

```bash
# npm / yarn
npm install socket.io-client mediasoup-client

# Flutter (add to pubspec.yaml)
socket_io_client: ^2.x.x
# Use a native WebRTC package + mediasoup-client-flutter if available
```

---

## Connection Setup

### Step 1 — Connect and register your userId

```js
const socket = io('http://your-server:8000');

socket.on('connect', () => {
  socket.emit('join', userId); // userId is a string, e.g. "42"
});
```

> You **must** emit `join` with your userId before doing anything else. The server uses this to route events to the right user.

---

## Socket Events Reference

### Events You Emit (Client → Server)

#### `join`
Register your user on connect.

```js
socket.emit('join', userId); // string
```

---

#### `group_call_initiate`
Start a group call in a conversation. The server automatically fetches all member info and conversation info from the database and notifies everyone.

```js
socket.emit('group_call_initiate', {
  callerId: '42',          // your user ID (string)
  conversationId: 'abc123', // the group conversation ID
  callType: 'video',        // 'video' | 'audio'
});
```

> After emitting this, the caller should immediately call `createRoom` (join MediaSoup room).

---

#### `createRoom` *(with callback)*
Join or create a MediaSoup room for the given conversation. Returns the router's RTP capabilities needed to create a MediaSoup Device.

```js
socket.emit('createRoom', { roomId: conversationId }, (response) => {
  if (response.error) { /* handle error */ return; }
  const { rtpCapabilities } = response;
  // Use rtpCapabilities to load your mediasoup-client Device
});
```

---

#### `createTransport` *(with callback)*
Create a WebRTC transport. Call this **twice** — once for sending, once for receiving.

```js
socket.emit('createTransport', { type: 'send' }, (res) => { /* use res to create sendTransport */ });
socket.emit('createTransport', { type: 'recv' }, (res) => { /* use res to create recvTransport */ });
```

Response shape:
```js
{
  id: string,
  iceParameters: object,
  iceCandidates: array,
  dtlsParameters: object,
}
```

---

#### `connectTransport` *(with callback)*
Connect a transport (triggered by the mediasoup-client `connect` event).

```js
socket.emit('connectTransport', {
  transportId: transport.id,
  dtlsParameters: dtlsParameters,
}, (res) => { /* res.success or res.error */ });
```

---

#### `produce` *(with callback)*
Start producing a media track (audio or video).

```js
socket.emit('produce', {
  transportId: sendTransport.id,
  kind: 'video',         // 'audio' | 'video'
  rtpParameters: rtpParameters,
}, (res) => {
  const { id } = res; // producer ID
});
```

---

#### `getProducers` *(with callback)*
Get a list of all existing producers in the room (for participants who joined mid-call).

```js
socket.emit('getProducers', (res) => {
  const { producers } = res;
  // producers: Array<{ id, kind, socketId, participantInfo }>
});
```

---

#### `consume` *(with callback)*
Start consuming a remote participant's media track.

```js
socket.emit('consume', {
  transportId: recvTransport.id,
  producerId: remoteProducerId,
  rtpCapabilities: device.rtpCapabilities,
}, (res) => {
  const { id, producerId, kind, rtpParameters } = res;
  // Create a consumer with recvTransport.consume(...)
});
```

---

#### `resumeConsumer` *(with callback)*
Resume a consumer after creation (required — consumers start paused).

```js
socket.emit('resumeConsumer', { consumerId: consumer.id }, (res) => { /* res.success */ });
```

---

#### `leaveRoom`
Leave the MediaSoup room. Call this when the user ends the call.

```js
socket.emit('leaveRoom');
```

---

### Events You Listen To (Server → Client)

#### `group_call_incoming`
Fired on all online group members (except the caller) when someone starts a call.

```js
socket.on('group_call_incoming', (data) => {
  // data.callerId          — string, user ID of the caller
  // data.conversationId    — string
  // data.callType          — 'video' | 'audio'
  // data.callerInfo        — { id, name, avatar }
  // data.conversationInfo  — { id, name, avatar }
});
```

Use `callerInfo` and `conversationInfo` to build your incoming call UI (name, avatar, group name).

---

#### `group_call_started`
Fired to **all members** of the conversation (including the caller) when the first person joins the MediaSoup room.

```js
socket.on('group_call_started', (data) => {
  // data.conversationId   — string
  // data.conversationInfo — { id, name, avatar }
  // data.callerInfo       — { id, name, avatar } (only present from group_call_initiate flow)
});
```

Use this to show a "Join call" banner for members who haven't joined yet.

---

#### `group_call_ended`
Fired to all members when the last person leaves the room.

```js
socket.on('group_call_ended', (data) => {
  // data.conversationId — string
  // Hide the "Join call" banner
});
```

---

#### `group_call_error`
Fired back to the caller if something went wrong (e.g. not a member, invalid ID).

```js
socket.on('group_call_error', (data) => {
  // data.message — string describing the error
});
```

---

#### `newProducer`
Fired to everyone in the room when a new participant starts sending media. Consume it immediately.

```js
socket.on('newProducer', async ({ producerId, kind, socketId, participantInfo }) => {
  // producerId      — string, the new producer's ID
  // kind            — 'audio' | 'video'
  // socketId        — string, the remote socket that produced it
  // participantInfo — { userId, name, avatar } — use to label the video tile
});
```

---

#### `participantLeft`
Fired to everyone in the room when someone leaves.

```js
socket.on('participantLeft', ({ socketId }) => {
  // Remove that participant's video tile from your UI
});
```

---

## Full Call Flow — Step by Step

### Caller Side

```
1. socket.emit('join', userId)
2. socket.emit('group_call_initiate', { callerId, conversationId, callType })
3. socket.emit('createRoom', { roomId: conversationId }, cb)
   → Load mediasoup Device with cb.rtpCapabilities
4. socket.emit('createTransport', { type: 'send' }, cb)  → create sendTransport
   socket.emit('createTransport', { type: 'recv' }, cb)  → create recvTransport
5. Get camera/mic stream via getUserMedia
6. sendTransport.produce(audioTrack)  → triggers 'produce' event → socket.emit('produce', ...)
   sendTransport.produce(videoTrack)  → same
7. socket.emit('getProducers', cb)  → consume existing participants' streams
8. Listen for 'newProducer' to consume participants who join later
```

### Receiver Side

```
1. socket.emit('join', userId)
2. Listen for 'group_call_incoming'
   → Show incoming call UI with callerInfo + conversationInfo
3. On "Accept":
   socket.emit('createRoom', { roomId: conversationId }, cb)
   → Follow steps 3–8 from Caller Side
4. On "Decline":
   → Simply dismiss the UI (no socket event needed for group calls)
```

### Joining a Call Already in Progress

```
1. Listen for 'group_call_started'  → show "Join call" banner
2. Listen for 'group_call_ended'    → hide "Join call" banner
3. On user taps "Join":
   socket.emit('createRoom', { roomId: conversationId }, cb)
   → Follow steps 3–8 from Caller Side
   → Use 'getProducers' to get all current streams
```

---

## MediaSoup WebRTC Setup

The complete handshake for setting up media:

```js
// 1. Load device
const device = new mediasoupClient.Device();
await device.load({ routerRtpCapabilities: rtpCapabilities });

// 2. Create send transport
const { id, iceParameters, iceCandidates, dtlsParameters } = await socketRequest('createTransport', { type: 'send' });
const sendTransport = device.createSendTransport({ id, iceParameters, iceCandidates, dtlsParameters });

sendTransport.on('connect', async ({ dtlsParameters }, callback, errback) => {
  await socketRequest('connectTransport', { transportId: sendTransport.id, dtlsParameters });
  callback();
});

sendTransport.on('produce', async ({ kind, rtpParameters }, callback, errback) => {
  const { id } = await socketRequest('produce', { transportId: sendTransport.id, kind, rtpParameters });
  callback({ id });
});

// 3. Create recv transport (same shape, type: 'recv')

// 4. Produce local tracks
const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true });
await sendTransport.produce({ track: stream.getAudioTracks()[0] });
await sendTransport.produce({ track: stream.getVideoTracks()[0] });

// 5. Consume a remote producer
const { id, producerId, kind, rtpParameters } = await socketRequest('consume', {
  transportId: recvTransport.id,
  producerId: remoteProducerId,
  rtpCapabilities: device.rtpCapabilities,
});
const consumer = await recvTransport.consume({ id, producerId, kind, rtpParameters });
await socketRequest('resumeConsumer', { consumerId: consumer.id });

// Attach consumer.track to a <video> element
videoElement.srcObject = new MediaStream([consumer.track]);
```

> `socketRequest` is a helper that wraps `socket.emit` in a Promise using the ack callback pattern.

---

## Participant Info & Conversation Info

The server automatically fetches names and avatars from the database. You never need to pass them manually. Here's where each piece of info arrives:

| Where it appears | Field | Source event |
|---|---|---|
| Incoming call UI | `callerInfo.name`, `callerInfo.avatar` | `group_call_incoming` |
| Incoming call UI | `conversationInfo.name`, `conversationInfo.avatar` | `group_call_incoming` |
| "Join call" banner | `conversationInfo.name`, `conversationInfo.avatar` | `group_call_started` |
| Video tile label | `participantInfo.name`, `participantInfo.avatar` | `newProducer` |
| Video tile label (late joiners) | `participantInfo` in each producer | `getProducers` callback |

### `callerInfo` shape

```json
{
  "id": 42,
  "name": "Alice",
  "avatar": "https://your-server/uploads/avatars/alice.jpg"
}
```

### `conversationInfo` shape

```json
{
  "id": "conv_abc123",
  "name": "Team Alpha",
  "avatar": "https://your-server/uploads/avatars/team_alpha.jpg"
}
```

> `avatar` can be `null` — always handle the fallback case (show initials or a placeholder).

### `participantInfo` shape (on producers)

```json
{
  "userId": "42",
  "name": "Alice",
  "avatar": "https://your-server/uploads/avatars/alice.jpg"
}
```

---

## Data Shapes

### `socketRequest` helper (recommended pattern)

```js
function socketRequest(event, data = {}) {
  return new Promise((resolve, reject) => {
    socket.emit(event, data, (response) => {
      if (response.error) reject(new Error(response.error));
      else resolve(response);
    });
  });
}
```

### Producer list item (from `getProducers`)

```ts
{
  id: string;            // mediasoup producer ID
  kind: 'audio' | 'video';
  socketId: string;      // which socket is producing
  participantInfo?: {
    userId: string;
    name: string;
    avatar: string | null;
  };
}
```

### `newProducer` event payload

```ts
{
  producerId: string;
  kind: 'audio' | 'video';
  socketId: string;
  participantInfo?: {
    userId: string;
    name: string;
    avatar: string | null;
  };
}
```

---

## Error Handling

| Error event | When it fires | What to do |
|---|---|---|
| `group_call_error` | Caller is not in the group, invalid ID, DB error | Show error message to user |
| `createRoom` callback `error: "Join first..."` | `join` was not emitted before `createRoom` | Emit `join` first, then retry |
| `createRoom` callback `error: "You are not in this group"` | User is not a member of the conversation | Do not proceed |
| `consume` callback `error: "RTP capabilities mismatch"` | Device not loaded yet | Wait for device load, retry |

---

## Edge Cases

**User joins mid-call:**
Call `getProducers` right after `createRoom` to get all existing streams. Each producer in the list includes `participantInfo` for labelling the video tile.

**Audio only / video only devices:**
The server accepts producers for whichever tracks exist. If the user has no camera, only produce audio. The other side handles missing video gracefully.

**Participant disconnects unexpectedly:**
The `participantLeft` event fires with the `socketId`. Remove the video tile for that socket.

**Avatar is null:**
Always check before setting an `<img src>`. Show the first letter of the name as a fallback.

**Call ends while a user is joining:**
`group_call_ended` fires to all members. If `currentRoomId` matches, clean up the call and return to the idle screen.

**Multiple sockets per user (user logged in on two devices):**
The server emits to the `userId` room, so both devices receive events. Each device must manage its own mediasoup state independently.

---

## Summary Diagram

```
Caller                    Server                    Other Members
  │                          │                           │
  │── join(userId) ─────────►│                           │
  │── group_call_initiate ──►│── group_call_incoming ───►│
  │                          │── group_call_started ─────►│ (all)
  │── createRoom ───────────►│                           │
  │◄─ rtpCapabilities ───────│                           │
  │── createTransport (send)►│                           │
  │── createTransport (recv)►│                           │
  │── connectTransport ─────►│                           │
  │── produce (audio) ──────►│── newProducer ────────────►│
  │── produce (video) ──────►│── newProducer ────────────►│
  │                          │                           │
  │                          │  [Member accepts]         │
  │                          │◄── createRoom ────────────│
  │                          │◄── produce (audio/video) ─│
  │◄─ newProducer ───────────│                           │
  │                          │                           │
  │── leaveRoom ────────────►│── participantLeft ─────────►│
  │                          │── group_call_ended (if empty)►│
```

## CODE
```js
    //==========================================mediasoup Group Call (room / SFU)===========================================
    socket.on(
      "createRoom",
      async (
        { roomId }: { roomId: string },
        cb: (arg: { rtpCapabilities?: mediasoup.types.RtpCapabilities; error?: string }) => void
      ) => {
        console.log("[createRoom] received", { roomId, socketId: socket.id });
        try {
          if (!roomId) {
            console.log("[createRoom] rejected: roomId required");
            cb({ error: "roomId required" });
            return;
          }
          const userId = getUserIdBySocket(socket.id);
          if (!userId) {
            console.log("[createRoom] rejected: user not joined (emit join with userId first)");
            cb({ error: "Join first with your user ID" });
            return;
          }
          console.log("[createRoom] checking membership for userId", userId);
          const [memberRow, conversation] = await Promise.all([
            fastify.prisma.conversationMember.findFirst({
              where: { conversationId: roomId, userId: Number(userId), isDeleted: false },
              select: {
                user: { select: { id: true, name: true, avatar: true } },
              },
            }),
            fastify.prisma.conversation.findUnique({
              where: { id: roomId },
              select: { id: true, name: true, avatar: true },
            }),
          ]);

          if (!memberRow) {
            console.log("[createRoom] rejected: user not in group", userId);
            cb({ error: "You are not in this group" });
            return;
          }

          const pInfo: ParticipantInfo = {
            userId,
            name: memberRow.user?.name ?? `User ${userId}`,
            avatar: memberRow.user?.avatar ? FileService.avatarUrl(memberRow.user.avatar) : null,
          };

          const conversationInfo = {
            id: roomId,
            name: conversation?.name ?? "Group Call",
            avatar: conversation?.avatar ? FileService.avatarUrl(conversation.avatar) : null,
          };

          const wasEmpty = isMediasoupRoomEmpty(roomId);
          console.log("[createRoom] room was empty:", wasEmpty, "→ creating/joining router");
          const router = await getOrCreateMediasoupRouter(roomId);
          mediasoupParticipants.set(socket.id, {
            roomId,
            router,
            transports: new Map(),
            producers: new Map(),
            consumers: new Map(),
            participantInfo: pInfo,
          });
          socket.join(roomId);
          if (wasEmpty) {
            console.log("[createRoom] first in room → emitting group_call_started to all members");
            const members = await fastify.prisma.conversationMember.findMany({
              where: { conversationId: roomId, isDeleted: false, userId: { not: null } },
              select: { userId: true },
            });
            for (const m of members) {
              if (m.userId) {
                io.to(String(m.userId)).emit("group_call_started", {
                  conversationId: roomId,
                  conversationInfo,
                });
              }
            }
          }
          console.log("[createRoom] success for socket", socket.id, "roomId", roomId);
          cb({ rtpCapabilities: router.rtpCapabilities });
        } catch (err: any) {
          console.error("[createRoom] failed", err);
          cb({ error: err?.message || "createRoom failed" });
        }
      }
    );

    socket.on(
      "createTransport",
      async (
        _payload: { type: string },
        cb: (arg: {
          id?: string;
          iceParameters?: any;
          iceCandidates?: any[];
          dtlsParameters?: any;
          error?: string;
        }) => void
      ) => {
        const type = _payload?.type ?? "?";
        console.log("[createTransport] received", { type, socketId: socket.id });
        try {
          const p = mediasoupParticipants.get(socket.id);
          if (!p) {
            console.log("[createTransport] rejected: participant not found");
            cb({ error: "Participant not found" });
            return;
          }
          const announcedIp =
            process.env.MEDIASOUP_ANNOUNCED_IP || getLocalIp();
          const transport = await p.router.createWebRtcTransport({
            listenIps: [{ ip: "0.0.0.0", announcedIp }],
            enableUdp: true,
            enableTcp: true,
            preferUdp: true,
            appData: { type: type === "recv" ? "recv" : "send" },
          });
          p.transports.set(transport.id, transport);
          transport.on("dtlsstatechange", (state) => {
            if (state === "closed") transport.close();
          });
          console.log("[createTransport] created", type, "transport", transport.id, "for socket", socket.id);
          cb({
            id: transport.id,
            iceParameters: transport.iceParameters,
            iceCandidates: transport.iceCandidates,
            dtlsParameters: transport.dtlsParameters,
          });
        } catch (err: any) {
          console.error("[createTransport] failed", err);
          cb({ error: err?.message || "createTransport failed" });
        }
      }
    );

    socket.on(
      "connectTransport",
      async (
        {
          transportId,
          dtlsParameters,
        }: { transportId: string; dtlsParameters: any },
        cb: (arg: { success?: boolean; error?: string }) => void
      ) => {
        console.log("[connectTransport] received", { transportId, socketId: socket.id });
        try {
          const p = mediasoupParticipants.get(socket.id);
          const t = p?.transports.get(transportId);
          if (!t) {
            console.log("[connectTransport] rejected: transport or participant not found");
            cb({ error: p ? "Transport not found" : "Participant not found" });
            return;
          }
          const transportType = (t.appData as { type?: string })?.type ?? "send";
          if (transportType === "recv") {
            // Recv transports: do not call connect(); they are ready after creation.
            // Client still sends connectTransport; we ack success so the client can proceed.
            console.log("[connectTransport] RECV transport", transportId, "— ack without connect()");
            cb({ success: true });
          } else {
            await t.connect({ dtlsParameters });
            console.log("[connectTransport] SEND transport", transportId, "connected");
            cb({ success: true });
          }
        } catch (err: any) {
          console.error("[connectTransport] failed", err);
          cb({ error: err?.message || "connectTransport failed" });
        }
      }
    );

    socket.on(
      "produce",
      async (
        {
          transportId,
          kind,
          rtpParameters,
        }: { transportId: string; kind: string; rtpParameters: any },
        cb: (arg: { id?: string; error?: string }) => void
      ) => {
        console.log("[produce] received", { transportId, kind, socketId: socket.id });
        try {
          const p = mediasoupParticipants.get(socket.id);
          const t = p?.transports.get(transportId);
          if (!t) {
            console.log("[produce] rejected: transport or participant not found");
            cb({ error: p ? "Transport not found" : "Participant not found" });
            return;
          }
          const producer = await t.produce({
            kind: kind as mediasoup.types.MediaKind,
            rtpParameters,
          });
          p!.producers.set(producer.id, producer);
          console.log("[produce] created producer", producer.id, kind, "→ emitting newProducer to room", p!.roomId);
          socket.to(p!.roomId).emit("newProducer", {
            producerId: producer.id,
            kind: producer.kind,
            socketId: socket.id,
            participantInfo: p!.participantInfo,
          });
          cb({ id: producer.id });
        } catch (err: any) {
          console.error("[produce] failed", err);
          cb({ error: err?.message || "produce failed" });
        }
      }
    );

    socket.on(
      "consume",
      async (
        {
          transportId,
          producerId,
          rtpCapabilities,
        }: { transportId: string; producerId: string; rtpCapabilities: any },
        cb: (arg: {
          id?: string;
          producerId?: string;
          kind?: string;
          rtpParameters?: any;
          error?: string;
        }) => void
      ) => {
        console.log("[consume] received", { transportId, producerId, socketId: socket.id });
        try {
          const p = mediasoupParticipants.get(socket.id);
          if (!p) {
            console.log("[consume] rejected: participant not found");
            cb({ error: "Participant not found" });
            return;
          }
          let producer: mediasoup.types.Producer | undefined;
          let producerSocketId: string | undefined;
          for (const [sid, px] of mediasoupParticipants) {
            if (px.producers.has(producerId)) {
              producer = px.producers.get(producerId);
              producerSocketId = sid;
              break;
            }
          }
          if (!producer || !producerSocketId) {
            console.log("[consume] rejected: producer not found", producerId);
            cb({ error: "Producer not found" });
            return;
          }
          if (producerSocketId === socket.id) {
            console.log("[consume] rejected: cannot consume own producer");
            cb({ error: "Cannot consume own producer" });
            return;
          }
          if (
            !p.router.canConsume({
              producerId,
              rtpCapabilities,
            })
          ) {
            console.log("[consume] rejected: RTP capabilities mismatch");
            cb({ error: "RTP capabilities mismatch" });
            return;
          }
          const recvTransport = p.transports.get(transportId);
          if (!recvTransport) {
            console.log("[consume] rejected: receive transport not found");
            cb({ error: "Receive transport not found" });
            return;
          }
          const consumer = await recvTransport.consume({
            producerId,
            rtpCapabilities,
            paused: false,
          });
          p.consumers.set(consumer.id, consumer);
          console.log("[consume] created consumer", consumer.id, "for producer", producerId, "socket", socket.id);
          cb({
            id: consumer.id,
            producerId: consumer.producerId,
            kind: consumer.kind,
            rtpParameters: consumer.rtpParameters,
          });
        } catch (err: any) {
          console.error("[consume] failed", err);
          cb({ error: err?.message || "consume failed" });
        }
      }
    );

    socket.on(
      "getProducers",
      (
        _dataOrCb: any,
        maybeCb?: (arg: { producers?: Array<{ id: string; kind: string; socketId: string }>; error?: string }) => void
      ) => {
        const cb =
          typeof _dataOrCb === "function" ? _dataOrCb : maybeCb;
        console.log("[getProducers] received", { socketId: socket.id });
        const p = mediasoupParticipants.get(socket.id);
        if (!p) {
          console.log("[getProducers] rejected: participant not found");
          cb?.({ error: "Participant not found" });
          return;
        }
        const producers = getMediasoupProducersForRoom(p.roomId);
        console.log("[getProducers] room", p.roomId, "producers count:", producers.length, producers);
        cb?.({ producers });
      }
    );

    socket.on(
      "resumeConsumer",
      async (
        { consumerId }: { consumerId: string },
        cb: (arg: { success?: boolean; error?: string }) => void
      ) => {
        console.log("[resumeConsumer] received", { consumerId, socketId: socket.id });
        try {
          const p = mediasoupParticipants.get(socket.id);
          const c = p?.consumers.get(consumerId);
          if (!c) {
            console.log("[resumeConsumer] rejected: consumer or participant not found");
            cb({ error: p ? "Consumer not found" : "Participant not found" });
            return;
          }
          if (c.paused) await c.resume();
          console.log("[resumeConsumer] resumed consumer", consumerId);
          cb({ success: true });
        } catch (err: any) {
          console.error("[resumeConsumer] failed", err);
          cb({ error: err?.message || "resumeConsumer failed" });
        }
      }
    );

    socket.on("leaveRoom", async () => {
      console.log("[leaveRoom] received", { socketId: socket.id });
      const roomId = cleanupMediasoupParticipant(socket.id);
      if (roomId) {
        socket.leave(roomId);
        console.log("[leaveRoom] socket left room", roomId, "→ emitting participantLeft");
        socket.to(roomId).emit("participantLeft", { socketId: socket.id });
        if (isMediasoupRoomEmpty(roomId)) {
          console.log("[leaveRoom] room now empty → emitting group_call_ended to all members");
          try {
            const members = await fastify.prisma.conversationMember.findMany({
              where: { conversationId: roomId, isDeleted: false, userId: { not: null } },
              select: { userId: true },
            });
            for (const m of members) {
              if (m.userId) io.to(String(m.userId)).emit("group_call_ended", { conversationId: roomId });
            }
          } catch (_) {}
        }
      } else {
        console.log("[leaveRoom] socket was not in any room");
      }
    });

    // Group call: verify caller is in group, fetch DB info, send push + socket to all members
    socket.on(
      "group_call_initiate",
      async ({
        callerId,
        conversationId,
        callType = "video",
      }: {
        callerId: string;
        conversationId: string;
        callType?: CallType;
      }) => {
        console.log("[group_call_initiate] received", {
          callerId,
          conversationId,
          callType,
          socketId: socket.id,
        });

        if (!callerId || !conversationId) {
          console.log("[group_call_initiate] skipped: missing callerId or conversationId");
          return;
        }

        const callerIdNumber = Number(callerId);
        if (Number.isNaN(callerIdNumber)) {
          socket.emit("group_call_error", { message: "Invalid caller id" });
          return;
        }

        try {
          // 1. Fetch conversation + members (with user FCM tokens + profile)
          const conversation = await fastify.prisma.conversation.findUnique({
            where: { id: conversationId },
            select: { id: true, name: true, avatar: true },
          });

          const members = await fastify.prisma.conversationMember.findMany({
            where: {
              conversationId,
              isDeleted: false,
              userId: { not: null },
            },
            select: {
              userId: true,
              user: {
                select: {
                  id: true,
                  name: true,
                  avatar: true,
                  fcmToken: true,
                },
              },
            },
          });

          const memberIds = members
            .map((m) => m.userId)
            .filter((id): id is number => id != null);
          console.log("[group_call_initiate] members", memberIds);

          // 2. Verify caller is a member
          const callerIsMember = memberIds.some((id) => String(id) === callerId);
          if (!callerIsMember) {
            console.log("[group_call_initiate] caller not in group");
            socket.emit("group_call_error", { message: "You are not in this group" });
            return;
          }

          // 3. Build caller info from DB
          const callerRow = members.find((m) => String(m.userId) === callerId)?.user;
          const callerInfo = {
            id: callerIdNumber,
            name: callerRow?.name ?? `User ${callerId}`,
            avatar: callerRow?.avatar ? FileService.avatarUrl(callerRow.avatar) : null,
          };

          // 4. Build conversation info
          const conversationInfo = {
            id: conversationId,
            name: conversation?.name ?? "Group Call",
            avatar: conversation?.avatar ? FileService.avatarUrl(conversation.avatar) : null,
          };

          console.log("[group_call_initiate] callerInfo", callerInfo);
          console.log("[group_call_initiate] conversationInfo", conversationInfo);

          // 5. Notify every other member: socket event + push notification
          const otherMembers = members.filter((m) => String(m.userId) !== callerId);
          console.log("[group_call_initiate] notifying members", otherMembers.map((m) => m.userId));

          for (const member of otherMembers) {
            if (!member.userId) continue;
            const userIdStr = String(member.userId);

            // Socket event (online users)
            const sockets = getSocketsForUser(userIdStr);
            if (sockets && sockets.size > 0) {
              console.log("[group_call_initiate] socket → group_call_incoming to", userIdStr);
              io.to(userIdStr).emit("group_call_incoming", {
                callerId,
                conversationId,
                callType,
                callerInfo,
                conversationInfo,
              });
            } else {
              console.log("[group_call_initiate] user", userIdStr, "offline – push only");
            }

            // Push notification (all members, including offline)
            const tokens = getJsonArray<string>(member.user?.fcmToken, []).filter(Boolean);
            if (tokens.length > 0) {
              const pushData: Record<string, string> = {
                type: "group_call_initiate",
                success: "true",
                title: conversationInfo.name,
                body: `${callerInfo.name} is calling`,
                message: `${callerInfo.name} started a group call`,
                conversationId,
                data: JSON.stringify({
                  callerId: String(callerId),
                  callType: String(callType),
                  conversationId,
                  callerInfo,
                  conversationInfo,
                }),
              };
              for (const token of tokens) {
                fastify.sendDataPush(token, pushData).catch((err: any) => {
                  console.warn("[group_call_initiate] push failed for", userIdStr, err?.message);
                });
              }
              console.log("[group_call_initiate] push sent to", userIdStr, `(${tokens.length} tokens)`);
            }
          }

          // 6. Tell every member (including caller) that a call is active so they can join
          console.log("[group_call_initiate] emitting group_call_started to all members");
          for (const m of members) {
            if (m.userId) {
              io.to(String(m.userId)).emit("group_call_started", {
                conversationId,
                callerInfo,
                conversationInfo,
              });
            }
          }
          console.log("[group_call_initiate] done");
        } catch (err: any) {
          console.error("[group_call_initiate] failed", err);
          fastify.log.warn(err, "group_call_initiate failed");
        }
      }
    );

    //==========================================mediasoup Group Call end===========================================

    // 13. Disconnect - Cleanup
    socket.on("disconnect", () => {
      const mediasoupRoomId = cleanupMediasoupParticipant(socket.id);
      if (mediasoupRoomId) {
        socket.to(mediasoupRoomId).emit("participantLeft", {
          socketId: socket.id,
        });
        if (isMediasoupRoomEmpty(mediasoupRoomId)) {
          void fastify.prisma.conversationMember
            .findMany({
              where: { conversationId: mediasoupRoomId, isDeleted: false, userId: { not: null } },
              select: { userId: true },
            })
            .then((members) => {
              for (const m of members) {
                if (m.userId) io.to(String(m.userId)).emit("group_call_ended", { conversationId: mediasoupRoomId });
              }
            })
            .catch(() => {});
        }
      }
```
