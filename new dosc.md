# Group Call — Events Quick Reference

> Clean reference table for frontend developers.  
> All events are Socket.IO. All IDs are `string` unless noted.

---

## Legend

| Symbol | Meaning |
|---|---|
| `→` | You **emit** this to the server |
| `←` | Server **emits** this to you |
| `⇄` | You emit with an **ack callback** — server replies via that callback |
| `*` | Field is optional / may be `null` |

---

## 1. Emit to Server (`→` and `⇄`)

| # | Event | Direction | Payload | Ack Response |
|---|---|---|---|---|
| 1 | `join` | `→` | `userId: string` | — |
| 2 | `get_active_calls` | `→` or `⇄` | `{}` *(empty object)* | `{ calls: ActiveCall[] }` or `{ error: string }` |
| 3 | `group_call_initiate` | `→` | `{ callerId, conversationId, callType }` | — |
| 4 | `createRoom` | `⇄` | `{ roomId: string }` | `{ rtpCapabilities }` or `{ error }` |
| 5 | `createTransport` | `⇄` | `{ type: "send" \| "recv" }` | `{ id, iceParameters, iceCandidates, dtlsParameters }` or `{ error }` |
| 6 | `connectTransport` | `⇄` | `{ transportId, dtlsParameters }` | `{ success: true }` or `{ error }` |
| 7 | `produce` | `⇄` | `{ transportId, kind, rtpParameters }` | `{ id: string }` or `{ error }` |
| 8 | `getProducers` | `⇄` | *(no payload)* | `{ producers: Producer[] }` |
| 9 | `consume` | `⇄` | `{ transportId, producerId, rtpCapabilities }` | `{ id, producerId, kind, rtpParameters }` or `{ error }` |
| 10 | `resumeConsumer` | `⇄` | `{ consumerId: string }` | `{ success: true }` or `{ error }` |
| 11 | `leaveRoom` | `→` | *(no payload)* | — |

---

## 2. Listen from Server (`←`)

| # | Event | Fires when | Payload |
|---|---|---|---|
| 1 | `active_group_calls` | After `join` (if calls exist), or response to `get_active_calls` without callback | `{ calls: ActiveCall[] }` |
| 2 | `group_call_incoming` | Someone starts a call in your group | `{ callerId, conversationId, callType, callerInfo, conversationInfo }` |
| 3 | `group_call_started` | First person joins the MediaSoup room | `{ conversationId, conversationInfo }` |
| 4 | `group_call_ended` | Last person leaves the room | `{ conversationId }` |
| 5 | `group_call_error` | Call initiation failed | `{ message: string }` |
| 6 | `newProducer` | Someone in the room starts sending audio/video | `{ producerId, kind, socketId, participantInfo* }` |
| 7 | `participantLeft` | Someone in the room disconnects or leaves | `{ socketId: string }` |
| 8 | `online-users` | Any user connects or disconnects | `string[]` — array of online userIds |

---

## 3. Payload Field Types

### `join` → payload
```
userId: string          // e.g. "42"
```

### `get_active_calls` → payload
```
{}                      // empty object, no fields needed
```

### `group_call_initiate` → payload
```
callerId:       string           // your user ID
conversationId: string           // the group conversation ID
callType:       "video" | "audio"
```

### `createRoom` ⇄ payload / response
```
// Send:
roomId: string

// Ack response (success):
rtpCapabilities: RtpCapabilities   // mediasoup RTP capabilities object

// Ack response (error):
error: string
```

### `createTransport` ⇄ payload / response
```
// Send:
type: "send" | "recv"

// Ack response (success):
id:             string
iceParameters:  object
iceCandidates:  object[]
dtlsParameters: object

// Ack response (error):
error: string
```

### `connectTransport` ⇄ payload / response
```
// Send:
transportId:    string
dtlsParameters: object   // from mediasoup-client transport 'connect' event

// Ack response:
success: true
// or
error: string
```

### `produce` ⇄ payload / response
```
// Send:
transportId:    string
kind:           "audio" | "video"
rtpParameters:  object   // from mediasoup-client transport 'produce' event

// Ack response:
id: string               // the new producer ID
// or
error: string
```

### `getProducers` ⇄ response
```
producers: Producer[]
```

### `consume` ⇄ payload / response
```
// Send:
transportId:    string
producerId:     string
rtpCapabilities: object  // device.rtpCapabilities

// Ack response (success):
id:             string
producerId:     string
kind:           "audio" | "video"
rtpParameters:  object

// Ack response (error):
error: string
```

### `resumeConsumer` ⇄ payload / response
```
// Send:
consumerId: string

// Ack response:
success: true
// or
error: string
```

---

## 4. Received Event Payload Types

### `active_group_calls`
```
calls: ActiveCall[]
```

### `group_call_incoming`
```
callerId:        string
conversationId:  string
callType:        "video" | "audio"
callerInfo:      CallerInfo
conversationInfo: ConversationInfo
```

### `group_call_started`
```
conversationId:  string
conversationInfo: ConversationInfo
```

### `group_call_ended`
```
conversationId: string
```

### `group_call_error`
```
message: string
```

### `newProducer`
```
producerId:      string
kind:            "audio" | "video"
socketId:        string
participantInfo: ParticipantInfo | undefined
```

### `participantLeft`
```
socketId: string
```

### `online-users`
```
string[]   // array of userId strings currently online
```

---

## 5. Shared Object Shapes

### `ActiveCall`
```ts
{
  conversationId:  string
  conversationInfo: ConversationInfo
  participantCount: number           // people currently in the call
}
```

### `ConversationInfo`
```ts
{
  id:     string
  name:   string
  avatar: string | null              // full URL — null if no image set
}
```

### `CallerInfo`
```ts
{
  id:     number                     // integer user ID
  name:   string
  avatar: string | null              // full URL — null if no image set
}
```

### `ParticipantInfo`
```ts
{
  userId: string
  name:   string
  avatar: string | null              // full URL — null if no image set
}
```

### `Producer` *(from `getProducers` response)*
```ts
{
  id:              string            // mediasoup producer ID
  kind:            "audio" | "video"
  socketId:        string
  participantInfo: ParticipantInfo | undefined
}
```

---

## 6. Event Flow at a Glance

```
ACTION                   YOU EMIT                 YOU RECEIVE
─────────────────────────────────────────────────────────────────
App opens / connect  →   join(userId)          ←  active_group_calls (if calls exist)
                                               ←  online-users

Someone starts call  ←                         ←  group_call_incoming
                                               ←  group_call_started

You start a call     →   group_call_initiate
You join the room    →   createRoom ⇄          ←  rtpCapabilities (ack)
                     →   createTransport x2 ⇄  ←  transport params (ack)
                     →   connectTransport ⇄    ←  success (ack)
                     →   produce (audio) ⇄     ←  producer id (ack)
                     →   produce (video) ⇄     ←  producer id (ack)
                     →   getProducers ⇄        ←  producers[] (ack)
                     →   consume (each) ⇄      ←  consumer params (ack)
                     →   resumeConsumer ⇄      ←  success (ack)

New person joins     ←                         ←  newProducer
                     →   consume ⇄             ←  consumer params (ack)
                     →   resumeConsumer ⇄      ←  success (ack)

Someone leaves       ←                         ←  participantLeft
Last person leaves   ←                         ←  group_call_ended

You leave            →   leaveRoom
                     ←                         ←  participantLeft (to others)
```

---

## 7. Error Responses

All `⇄` (ack) events return `{ error: string }` on failure. Always check before using the response.

```js
socket.emit('createRoom', { roomId }, (res) => {
  if (res.error) {
    console.error(res.error);
    return;
  }
  // safe to use res.rtpCapabilities
});
```

| Event | Possible error values |
|---|---|
| `createRoom` | `"roomId required"`, `"Join first with your user ID"`, `"You are not in this group"` |
| `createTransport` | `"Participant not found"` |
| `connectTransport` | `"Participant not found"`, `"Transport not found"` |
| `produce` | `"Participant not found"`, `"Transport not found"` |
| `consume` | `"Participant not found"`, `"Producer not found"`, `"Cannot consume own producer"`, `"RTP capabilities mismatch"`, `"Receive transport not found"` |
| `resumeConsumer` | `"Participant not found"`, `"Consumer not found"` |
| `get_active_calls` | `"Not joined"`, `"Invalid user id"` |
| `group_call_error` event | `"You are not in this group"`, `"Invalid caller id"` |



## BACKEND CODE
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





## HTML CODE FOR TEST
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Group Call (Threads)</title>
    <script src="https://cdn.socket.io/4.8.3/socket.io.min.js"></script>
    <script type="module">
        // Load mediasoup-client using Skypack (reliable ES module CDN)
        import * as mediasoupClient from 'https://cdn.skypack.dev/mediasoup-client@3.6.92';
        window.mediasoupClient = mediasoupClient;
        console.log('✅ mediasoup-client loaded via Skypack');
        
        // Also set a flag so other scripts know it's ready
        window.mediasoupClientReady = true;
        window.dispatchEvent(new Event('mediasoupClientReady'));
    </script>
    <script>
        // Wait for mediasoup-client to be ready (for non-module code)
        if (!window.mediasoupClient) {
            window.addEventListener('mediasoupClientReady', function() {
                console.log('mediasoup-client is now available');
            });
        }
    </script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            color: #333;
        }

        .container {
            max-width: 1400px;
            margin: 0 auto;
            padding: 20px;
        }

        .header {
            background: white;
            border-radius: 12px;
            padding: 20px;
            margin-bottom: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }

        .header h1 {
            color: #667eea;
            margin-bottom: 10px;
        }

        .status {
            display: inline-block;
            padding: 6px 12px;
            border-radius: 20px;
            font-size: 14px;
            font-weight: 500;
            margin-top: 10px;
        }

        .status.connected {
            background: #10b981;
            color: white;
        }

        .status.disconnected {
            background: #ef4444;
            color: white;
        }

        .status.in-call {
            background: #3b82f6;
            color: white;
        }

        .setup-section {
            background: white;
            border-radius: 12px;
            padding: 30px;
            margin-bottom: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }

        .setup-section h2 {
            color: #667eea;
            margin-bottom: 20px;
        }

        .form-group {
            margin-bottom: 20px;
        }

        .form-group label {
            display: block;
            margin-bottom: 8px;
            font-weight: 500;
            color: #555;
        }

        .form-group input,
        .form-group select {
            width: 100%;
            padding: 12px;
            border: 2px solid #e5e7eb;
            border-radius: 8px;
            font-size: 16px;
            transition: border-color 0.3s;
        }

        .form-group input:focus,
        .form-group select:focus {
            outline: none;
            border-color: #667eea;
        }

        .btn {
            padding: 12px 24px;
            border: none;
            border-radius: 8px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.3s;
            margin-right: 10px;
            margin-top: 10px;
        }

        .btn-primary {
            background: #667eea;
            color: white;
        }

        .btn-primary:hover {
            background: #5568d3;
            transform: translateY(-2px);
            box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
        }

        .btn-danger {
            background: #ef4444;
            color: white;
        }

        .btn-danger:hover {
            background: #dc2626;
        }

        .btn-secondary {
            background: #6b7280;
            color: white;
        }

        .btn-secondary:hover {
            background: #4b5563;
        }

        .btn:disabled {
            opacity: 0.5;
            cursor: not-allowed;
        }

        .users-list {
            background: white;
            border-radius: 12px;
            padding: 20px;
            margin-bottom: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            max-height: 300px;
            overflow-y: auto;
        }

        .users-list h3 {
            color: #667eea;
            margin-bottom: 15px;
        }

        .user-item {
            padding: 12px;
            border: 1px solid #e5e7eb;
            border-radius: 8px;
            margin-bottom: 10px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .user-item:hover {
            background: #f9fafb;
        }

        .video-section {
            background: white;
            border-radius: 12px;
            padding: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            display: none;
        }

        .video-section.active {
            display: block;
        }

        .video-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 20px;
            margin-bottom: 20px;
        }

        .video-container {
            position: relative;
            background: #000;
            border-radius: 12px;
            overflow: hidden;
            aspect-ratio: 16/9;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
        }

        .video-container video {
            width: 100%;
            height: 100%;
            object-fit: cover;
            background: #000;
            display: block;
        }
        
        .video-container video:not([src]):not([srcObject]) {
            background: #1a1a1a;
        }

        .video-label {
            position: absolute;
            bottom: 10px;
            left: 10px;
            background: rgba(0, 0, 0, 0.7);
            color: white;
            padding: 6px 12px;
            border-radius: 6px;
            font-size: 14px;
            font-weight: 500;
        }

        .video-controls {
            position: absolute;
            bottom: 10px;
            right: 10px;
            display: flex;
            gap: 10px;
        }

        .control-btn {
            width: 40px;
            height: 40px;
            border-radius: 50%;
            border: none;
            background: rgba(255, 255, 255, 0.2);
            color: white;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 18px;
            transition: all 0.3s;
            backdrop-filter: blur(10px);
        }

        .control-btn:hover {
            background: rgba(255, 255, 255, 0.3);
            transform: scale(1.1);
        }

        .control-btn.muted {
            background: #ef4444;
        }

        .control-btn.video-off {
            background: #6b7280;
        }

        .controls-bar {
            display: flex;
            justify-content: center;
            gap: 15px;
            padding: 20px;
            background: #f9fafb;
            border-radius: 12px;
            margin-top: 20px;
        }

        .controls-bar .btn {
            min-width: 120px;
        }

        .error {
            background: #fee2e2;
            color: #991b1b;
            padding: 12px;
            border-radius: 8px;
            margin-top: 10px;
            display: none;
        }

        .error.show {
            display: block;
        }

        .loading {
            text-align: center;
            padding: 20px;
            color: #6b7280;
        }

        .incoming-call {
            background: white;
            border-radius: 16px;
            padding: 24px;
            margin-bottom: 20px;
            box-shadow: 0 8px 24px rgba(102, 126, 234, 0.18);
            border: 2px solid #667eea33;
            animation: ring-pulse 1.2s ease-in-out infinite;
        }

        @keyframes ring-pulse {
            0%   { box-shadow: 0 8px 24px rgba(102,126,234,0.18); }
            50%  { box-shadow: 0 8px 40px rgba(102,126,234,0.42); }
            100% { box-shadow: 0 8px 24px rgba(102,126,234,0.18); }
        }

        .incoming-call h3 {
            color: #667eea;
            margin-bottom: 12px;
            font-size: 18px;
        }

        .incoming-call p {
            margin-bottom: 16px;
            color: #555;
        }

        .group-call-join {
            background: linear-gradient(135deg, #10b981 0%, #059669 100%);
            color: white;
            border-radius: 12px;
            padding: 16px 20px;
            margin-bottom: 20px;
            display: none;
            align-items: center;
            justify-content: space-between;
            box-shadow: 0 4px 12px rgba(16, 185, 129, 0.3);
        }

        .group-call-join.active {
            display: flex;
        }

        .group-call-join .join-info {
            display: flex;
            align-items: center;
            gap: 12px;
        }

        .group-call-join .join-avatar {
            width: 40px;
            height: 40px;
            border-radius: 50%;
            object-fit: cover;
            border: 2px solid rgba(255,255,255,0.5);
            display: none;
        }

        .group-call-join .join-avatar-fallback {
            width: 40px;
            height: 40px;
            border-radius: 50%;
            background: rgba(255,255,255,0.25);
            color: white;
            font-size: 16px;
            font-weight: 700;
            display: none;
            align-items: center;
            justify-content: center;
            border: 2px solid rgba(255,255,255,0.5);
            flex-shrink: 0;
        }

        .group-call-join span {
            font-weight: 600;
        }

        .group-call-join .btn-join {
            padding: 8px 20px;
            background: white;
            color: #059669;
            border: none;
            border-radius: 8px;
            font-weight: 600;
            cursor: pointer;
            flex-shrink: 0;
        }

        .group-call-join .btn-join:hover {
            background: #f0fdf4;
        }

        /* Call header inside the video section */
        .call-header {
            display: flex;
            align-items: center;
            gap: 14px;
            margin-bottom: 20px;
            padding-bottom: 16px;
            border-bottom: 1px solid #e5e7eb;
        }

        .call-header-avatar {
            width: 48px;
            height: 48px;
            border-radius: 50%;
            object-fit: cover;
            border: 2px solid #667eea33;
            display: none;
        }

        .call-header-avatar-fallback {
            width: 48px;
            height: 48px;
            border-radius: 50%;
            background: #667eea;
            color: white;
            font-size: 20px;
            font-weight: 700;
            display: none;
            align-items: center;
            justify-content: center;
            flex-shrink: 0;
        }

        .call-header-info h2 {
            color: #667eea;
            font-size: 20px;
            margin: 0;
        }

        .call-header-info p {
            color: #6b7280;
            font-size: 13px;
            margin: 2px 0 0;
        }

        /* Participant avatar in video tile */
        .video-participant-info {
            position: absolute;
            bottom: 0;
            left: 0;
            right: 0;
            padding: 28px 10px 10px;
            background: linear-gradient(to top, rgba(0,0,0,0.75) 0%, transparent 100%);
            display: flex;
            align-items: flex-end;
            gap: 8px;
        }

        .participant-avatar-wrap {
            flex-shrink: 0;
        }

        .participant-avatar {
            width: 32px;
            height: 32px;
            border-radius: 50%;
            object-fit: cover;
            border: 2px solid rgba(255,255,255,0.5);
            display: none;
        }

        .participant-avatar-fallback {
            width: 32px;
            height: 32px;
            border-radius: 50%;
            background: #667eea;
            color: white;
            font-size: 13px;
            font-weight: 700;
            display: none;
            align-items: center;
            justify-content: center;
            border: 2px solid rgba(255,255,255,0.5);
        }

        .participant-name {
            color: white;
            font-size: 13px;
            font-weight: 600;
            text-shadow: 0 1px 3px rgba(0,0,0,0.6);
        }

        @media (max-width: 768px) {
            .video-grid {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🎥 Group Call (Threads)</h1>
            <div class="status disconnected" id="status">Disconnected</div>
        </div>

        <div class="setup-section" id="setupSection">
            <h2>Setup</h2>
            <div class="form-group">
                <label for="serverUrl">Server URL</label>
                <input type="text" id="serverUrl" placeholder="http://localhost:8000" value="http://localhost:8000">
            </div>
            <div class="form-group">
                <label for="userId">Your User ID</label>
                <input type="text" id="userId" placeholder="e.g. 1" value="1">
            </div>
            <div class="form-group">
                <label for="username">Your Name</label>
                <input type="text" id="username" placeholder="Display name" value="User">
            </div>
            <div class="form-group">
                <label for="conversationId">Conversation ID (group)</label>
                <input type="text" id="conversationId" placeholder="Same for all participants" value="">
            </div>
            <div class="form-group">
                <label for="audioDeviceSelect">Microphone</label>
                <select id="audioDeviceSelect">
                    <option value="">Default</option>
                </select>
            </div>
            <div class="form-group">
                <label for="videoDeviceSelect">Camera</label>
                <select id="videoDeviceSelect">
                    <option value="">Default</option>
                </select>
            </div>
            <button class="btn btn-secondary" onclick="refreshDeviceList()" style="margin-bottom: 10px;">Refresh devices</button>
            <button class="btn btn-primary" onclick="connect()">Connect</button>
            <button class="btn btn-primary" id="btnStartGroupCall" onclick="startGroupCall()" disabled>Start group call</button>
            <div class="error" id="error"></div>
        </div>

        <div class="incoming-call" id="incomingCallSection" style="display: none;">
            <h3>📞 Incoming Group Call</h3>
            <div style="display:flex;align-items:center;gap:14px;margin:14px 0;">
                <img id="incomingConvAvatar" src="" alt=""
                     style="width:52px;height:52px;border-radius:50%;object-fit:cover;background:#e5e7eb;display:none;">
                <div style="display:inline-block;width:52px;height:52px;border-radius:50%;background:#667eea;color:#fff;font-size:22px;font-weight:700;text-align:center;line-height:52px;display:none;" id="incomingConvAvatarFallback"></div>
                <div>
                    <div style="font-weight:700;font-size:16px;color:#111;" id="incomingConvName"></div>
                    <div style="font-size:13px;color:#6b7280;">Group call</div>
                </div>
            </div>
            <div style="display:flex;align-items:center;gap:10px;margin-bottom:16px;">
                <img id="incomingCallerAvatar" src="" alt=""
                     style="width:36px;height:36px;border-radius:50%;object-fit:cover;background:#e5e7eb;display:none;">
                <div style="display:inline-block;width:36px;height:36px;border-radius:50%;background:#10b981;color:#fff;font-size:14px;font-weight:700;text-align:center;line-height:36px;display:none;" id="incomingCallerAvatarFallback"></div>
                <span style="color:#374151;" id="incomingCallFrom"></span>
            </div>
            <div style="display:flex;gap:12px;">
                <button class="btn btn-primary" onclick="acceptGroupCall()">✅ Accept</button>
                <button class="btn btn-danger" onclick="declineGroupCall()">❌ Decline</button>
            </div>
        </div>

        <div class="group-call-join" id="groupCallJoinSection">
            <div class="join-info">
                <img id="joinConvAvatar" class="join-avatar" src="" alt="">
                <div class="join-avatar-fallback" id="joinConvAvatarFallback"></div>
                <span id="joinConvLabel">📞 Group call in progress – you can join</span>
            </div>
            <button type="button" class="btn-join" id="btnJoinGroupCall" onclick="joinActiveGroupCall()">Join</button>
        </div>

        <div class="video-section" id="videoSection">
            <div class="call-header">
                <img id="callHeaderAvatar" class="call-header-avatar" src="" alt="">
                <div class="call-header-avatar-fallback" id="callHeaderAvatarFallback"></div>
                <div class="call-header-info">
                    <h2 id="callHeaderTitle">Group Call</h2>
                    <p id="callHeaderSub">In call</p>
                </div>
            </div>
            <div class="video-grid" id="videoGrid"></div>
            <div class="controls-bar">
                <button class="btn btn-primary" id="toggleAudio" onclick="toggleAudio()">
                    🔊 Unmute
                </button>
                <button class="btn btn-primary" id="toggleVideo" onclick="toggleVideo()">
                    📹 Video On
                </button>
                <button class="btn btn-danger" onclick="leaveRoom()">Leave Call</button>
            </div>
        </div>
    </div>

    <script>
        // Configuration — threads backend (group call)
        function getServerUrl() {
            const el = document.getElementById('serverUrl');
            return (el && el.value && el.value.trim()) ? el.value.trim() : 'http://localhost:8000';
        }
        let socket = null;
        let device = null;
        let pendingGroupCall = null; // { callerId, conversationId, callType, callerInfo }
        let activeGroupCallConversationId = null; // conversationId that has an active call (show "you can join")
        let activeConversationInfo = null; // { id, name, avatar } for the current active group call
        let socketParticipantInfo = new Map(); // socketId → { userId, name, avatar }

        // Wait for mediasoup-client to load
        function waitForMediasoupClient() {
            return new Promise((resolve, reject) => {
                // If already ready, resolve immediately
                if (window.mediasoupClient && window.mediasoupClient.Device) {
                    resolve();
                    return;
                }
                
                // Wait for the ready event
                const onReady = () => {
                    if (window.mediasoupClient && window.mediasoupClient.Device) {
                        resolve();
                    }
                };
                
                window.addEventListener('mediasoupClientReady', onReady);
                
                // Also poll as fallback
                let attempts = 0;
                const maxAttempts = 100; // 10 seconds
                
                const check = () => {
                    const mediasoup = window.mediasoupClient;
                    if (mediasoup && mediasoup.Device) {
                        window.removeEventListener('mediasoupClientReady', onReady);
                        resolve();
                    } else if (attempts < maxAttempts) {
                        attempts++;
                        setTimeout(check, 100);
                    } else {
                        window.removeEventListener('mediasoupClientReady', onReady);
                        reject(new Error(
                            'mediasoup-client failed to load after 10 seconds. ' +
                            'This might be due to:\n' +
                            '1. Network connectivity issues\n' +
                            '2. CORS restrictions\n' +
                            '3. Browser blocking the CDN\n\n' +
                            'Please refresh the page or check the browser console.'
                        ));
                    }
                };
                
                check();
            });
        }
        let sendTransport = null;
        let recvTransport = null;
        let recvTransportConnected = false; // Track recv transport connection state
        let producers = new Map();
        let consumers = new Map();
        let producerToSocket = new Map(); // Track which producer belongs to which socket
        let socketToStream = new Map(); // Track streams per socket (to combine audio/video)
        let localStream = null;
        let currentRoomId = null;
        let isAudioEnabled = true;
        let isVideoEnabled = true;

        // Helper function to check device availability
        async function checkDeviceAvailability() {
            try {
                const devices = await navigator.mediaDevices.enumerateDevices();
                const hasVideo = devices.some(d => d.kind === 'videoinput');
                const hasAudio = devices.some(d => d.kind === 'audioinput');
                return { hasVideo, hasAudio, devices };
            } catch (error) {
                console.warn('Could not enumerate devices:', error);
                return { hasVideo: true, hasAudio: true, devices: [] };
            }
        }

        // Populate microphone and camera dropdowns (flexible: works with one or many devices)
        async function refreshDeviceList() {
            const audioSelect = document.getElementById('audioDeviceSelect');
            const videoSelect = document.getElementById('videoDeviceSelect');
            if (!audioSelect || !videoSelect) return;
            try {
                let devices = await navigator.mediaDevices.enumerateDevices();
                // If labels are empty, get permission first so we can show names
                if (devices.length > 0 && !devices[0].label) {
                    try {
                        const tempStream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true });
                        tempStream.getTracks().forEach(t => t.stop());
                        devices = await navigator.mediaDevices.enumerateDevices();
                    } catch (_) { /* use devices as-is */ }
                }
                const audioDevices = devices.filter(d => d.kind === 'audioinput');
                const videoDevices = devices.filter(d => d.kind === 'videoinput');
                audioSelect.innerHTML = '<option value="">Default</option>';
                audioDevices.forEach((d, i) => {
                    const opt = document.createElement('option');
                    opt.value = d.deviceId;
                    opt.textContent = d.label || `Microphone ${i + 1}`;
                    audioSelect.appendChild(opt);
                });
                videoSelect.innerHTML = '<option value="">Default</option>';
                videoDevices.forEach((d, i) => {
                    const opt = document.createElement('option');
                    opt.value = d.deviceId;
                    opt.textContent = d.label || `Camera ${i + 1}`;
                    videoSelect.appendChild(opt);
                });
                // If only one device, preselect it
                if (audioDevices.length === 1) audioSelect.value = audioDevices[0].deviceId;
                if (videoDevices.length === 1) videoSelect.value = videoDevices[0].deviceId;
            } catch (e) {
                console.warn('Could not refresh device list:', e);
            }
        }

        function getUserId() {
            return (document.getElementById('userId') && document.getElementById('userId').value) || '';
        }

        function getConversationId() {
            return (document.getElementById('conversationId') && document.getElementById('conversationId').value) || '';
        }

        function updateGroupCallJoinVisibility() {
            const el = document.getElementById('groupCallJoinSection');
            if (!el) return;
            const convId = getConversationId();
            const alreadyInCall = currentRoomId === activeGroupCallConversationId;
            if (activeGroupCallConversationId && convId && activeGroupCallConversationId === convId && !alreadyInCall) {
                el.classList.add('active');
            } else {
                el.classList.remove('active');
            }
        }

        function joinActiveGroupCall() {
            if (!activeGroupCallConversationId) return;
            joinRoom(activeGroupCallConversationId);
            document.getElementById('groupCallJoinSection').classList.remove('active');
        }

        // Connect to backend and register as user (do not join room yet)
        async function connect() {
            try {
                const userId = getUserId();
                if (!userId) {
                    showError('Please enter your User ID');
                    return;
                }

                // Clean up any existing streams
                if (localStream) {
                    localStream.getTracks().forEach(track => track.stop());
                    localStream = null;
                }

                const deviceCheck = await checkDeviceAvailability();
                if (!deviceCheck.hasVideo && !deviceCheck.hasAudio) {
                    showError('No camera or microphone found. Please connect a device and refresh the page.');
                    return;
                }

                socket = io(getServerUrl());
                updateStatus('connecting');

                socket.on('connect', () => {
                    console.log('Socket connected');
                    socket.emit('join', userId);
                    updateStatus('connected');
                    document.getElementById('btnStartGroupCall').disabled = false;
                });

                socket.on('disconnect', () => {
                    console.log('Socket disconnected');
                    updateStatus('disconnected');
                    document.getElementById('btnStartGroupCall').disabled = true;
                });

                socket.on('group_call_error', (data) => {
                    showError(data.message || 'Group call failed');
                });

                socket.on('group_call_incoming', (data) => {
                    pendingGroupCall = data;

                    // Conversation info
                    const conv = data.conversationInfo || {};
                    const convName = conv.name || 'Group Call';
                    document.getElementById('incomingConvName').textContent = convName;
                    const convAvatarEl = document.getElementById('incomingConvAvatar');
                    const convFallbackEl = document.getElementById('incomingConvAvatarFallback');
                    if (conv.avatar) {
                        convAvatarEl.src = conv.avatar;
                        convAvatarEl.style.display = 'block';
                        convFallbackEl.style.display = 'none';
                    } else {
                        convAvatarEl.style.display = 'none';
                        convFallbackEl.textContent = convName.charAt(0).toUpperCase();
                        convFallbackEl.style.display = 'inline-block';
                    }

                    // Caller info
                    const caller = data.callerInfo || {};
                    const callerName = caller.name || `User ${data.callerId}`;
                    document.getElementById('incomingCallFrom').textContent = `${callerName} is calling`;
                    const callerAvatarEl = document.getElementById('incomingCallerAvatar');
                    const callerFallbackEl = document.getElementById('incomingCallerAvatarFallback');
                    if (caller.avatar) {
                        callerAvatarEl.src = caller.avatar;
                        callerAvatarEl.style.display = 'block';
                        callerFallbackEl.style.display = 'none';
                    } else {
                        callerAvatarEl.style.display = 'none';
                        callerFallbackEl.textContent = callerName.charAt(0).toUpperCase();
                        callerFallbackEl.style.display = 'inline-block';
                    }

                    document.getElementById('incomingCallSection').style.display = 'block';
                });

                socket.on('group_call_started', (data) => {
                    activeGroupCallConversationId = data.conversationId || null;
                    if (data.conversationInfo) {
                        activeConversationInfo = data.conversationInfo;
                    }
                    updateGroupCallJoinVisibility();
                    // Update join banner with conversation name + avatar
                    const conv = data.conversationInfo || {};
                    const convName = conv.name || 'Group Call';
                    const joinLabel = document.getElementById('joinConvLabel');
                    if (joinLabel) {
                        joinLabel.textContent = `📞 "${convName}" call in progress – you can join`;
                    }
                    const joinAvatar = document.getElementById('joinConvAvatar');
                    const joinFallback = document.getElementById('joinConvAvatarFallback');
                    if (joinAvatar && joinFallback) {
                        if (conv.avatar) {
                            joinAvatar.src = conv.avatar;
                            joinAvatar.style.display = 'block';
                            joinFallback.style.display = 'none';
                        } else {
                            joinAvatar.style.display = 'none';
                            joinFallback.textContent = convName.charAt(0).toUpperCase();
                            joinFallback.style.display = 'flex';
                        }
                    }
                });

                socket.on('group_call_ended', (data) => {
                    if (activeGroupCallConversationId === (data.conversationId || null)) {
                        activeGroupCallConversationId = null;
                    }
                    updateGroupCallJoinVisibility();
                });

                socket.on('newProducer', async ({ producerId, kind, socketId, participantInfo }) => {
                    if (!socket || socketId === socket.id) return;
                    if (participantInfo) {
                        socketParticipantInfo.set(socketId, participantInfo);
                    }
                    let retries = 0;
                    while ((!recvTransport || !device || !device.loaded) && retries < 15) {
                        await new Promise(resolve => setTimeout(resolve, 100));
                        retries++;
                    }
                    if (!recvTransport || !device || !device.loaded) {
                        console.error('Not ready for producer ' + producerId);
                        return;
                    }
                    producerToSocket.set(producerId, socketId);
                    try {
                        await consumeProducer(producerId, kind);
                    } catch (error) {
                        console.error('Failed to consume producer ' + producerId, error);
                    }
                });

                socket.on('participantLeft', ({ socketId }) => {
                    removeParticipant(socketId);
                });

            } catch (error) {
                console.error('Connection error:', error);
                showError('Failed to connect: ' + error.message);
            }
        }

        // Start group call: server fetches caller + conversation info from DB automatically
        async function startGroupCall() {
            const userId = getUserId();
            const conversationId = getConversationId();
            if (!userId || !conversationId) {
                showError('Enter User ID and Conversation ID');
                return;
            }
            // activeConversationInfo will be populated when group_call_started arrives;
            // set a placeholder so the header is at least correct while waiting
            if (!activeConversationInfo || activeConversationInfo.id !== conversationId) {
                activeConversationInfo = { id: conversationId, name: 'Group Call', avatar: null };
            }
            socket.emit('group_call_initiate', {
                callerId: userId,
                conversationId: conversationId,
                callType: 'video',
            });
            await joinRoom(conversationId);
        }

        async function acceptGroupCall() {
            if (!pendingGroupCall) return;
            const conversationId = pendingGroupCall.conversationId;
            if (pendingGroupCall.conversationInfo) {
                activeConversationInfo = pendingGroupCall.conversationInfo;
            }
            document.getElementById('incomingCallSection').style.display = 'none';
            pendingGroupCall = null;
            await joinRoom(conversationId);
        }

        function declineGroupCall() {
            document.getElementById('incomingCallSection').style.display = 'none';
            pendingGroupCall = null;
        }

        function updateCallHeader(convInfo) {
            const conv = convInfo || activeConversationInfo || {};
            const name = conv.name || 'Group Call';
            const titleEl = document.getElementById('callHeaderTitle');
            const subEl = document.getElementById('callHeaderSub');
            const avatarEl = document.getElementById('callHeaderAvatar');
            const fallbackEl = document.getElementById('callHeaderAvatarFallback');
            if (titleEl) titleEl.textContent = name;
            if (subEl) subEl.textContent = 'In call';
            if (avatarEl && fallbackEl) {
                if (conv.avatar) {
                    avatarEl.src = conv.avatar;
                    avatarEl.style.display = 'block';
                    fallbackEl.style.display = 'none';
                } else {
                    avatarEl.style.display = 'none';
                    fallbackEl.textContent = name.charAt(0).toUpperCase();
                    fallbackEl.style.display = 'flex';
                }
            }
        }

        async function joinRoom(roomId) {
            try {
                currentRoomId = roomId;
                updateStatus('in-call');
                socketParticipantInfo.clear();

                // Wait for mediasoup-client to be available
                await waitForMediasoupClient();

                // Create or join room
                const { rtpCapabilities } = await socketRequest('createRoom', { roomId });
                console.log('Room joined, RTP capabilities:', rtpCapabilities);

                // Get mediasoupClient reference
                const mediasoup = window.mediasoupClient || mediasoupClient || window.mediasoup;
                if (!mediasoup) {
                    throw new Error('mediasoup-client library not loaded. Please refresh the page.');
                }

                // Create mediasoup device
                device = new mediasoup.Device();
                await device.load({ routerRtpCapabilities: rtpCapabilities });
                console.log('Device loaded');

                // Create transports in parallel for faster connection
                await Promise.all([
                    createSendTransport(),
                    createRecvTransport()
                ]);

                // Optimized: Minimal wait (100ms instead of 800ms)
                await new Promise(resolve => setTimeout(resolve, 100));

                // Start producing local media
                await produceMedia();
                
                // Optimized: Minimal wait (100ms instead of 500ms)
                await new Promise(resolve => setTimeout(resolve, 100));

                // Get existing producers and consume in parallel
                const { producers: existingProducers } = await socketRequest('getProducers');

                // Consume existing producers in parallel for faster connection
                if (recvTransport && device && device.loaded) {
                    const consumePromises = existingProducers
                        .filter(producer => producer.socketId !== socket.id)
                        .map(async (producer) => {
                            if (producer.participantInfo) {
                                socketParticipantInfo.set(producer.socketId, producer.participantInfo);
                            }
                            producerToSocket.set(producer.id, producer.socketId);
                            try {
                                await consumeProducer(producer.id, producer.kind);
                            } catch (error) {
                                console.error(`❌ Failed to consume producer ${producer.id}:`, error);
                            }
                        });
                    await Promise.all(consumePromises);
                }

                // Initialize button states
                document.getElementById('toggleAudio').textContent = '🔊 Unmute';
                document.getElementById('toggleVideo').textContent = '📹 Video On';

                // Show video section with conversation header
                updateCallHeader(activeConversationInfo);
                document.getElementById('setupSection').style.display = 'none';
                document.getElementById('videoSection').classList.add('active');
                document.getElementById('incomingCallSection').style.display = 'none';

            } catch (error) {
                console.error('Join room error:', error);
                showError('Failed to join room: ' + error.message);
            }
        }

        async function createSendTransport() {
            const { id, iceParameters, iceCandidates, dtlsParameters } = await socketRequest('createTransport', { type: 'send' });

            sendTransport = device.createSendTransport({
                id,
                iceParameters,
                iceCandidates,
                dtlsParameters,
            });

            sendTransport.on('connect', async ({ dtlsParameters }, callback, errback) => {
                try {
                    console.log('Connecting send transport...');
                    await socketRequest('connectTransport', {
                        transportId: sendTransport.id,
                        dtlsParameters,
                    });
                    console.log('✅ Send transport connected');
                    callback();
                } catch (error) {
                    console.error('❌ Send transport connection error:', error);
                    errback(error);
                }
            });

            sendTransport.on('connectionstatechange', (state) => {
                console.log('Send transport connection state:', state);
            });

            sendTransport.on('produce', async ({ kind, rtpParameters }, callback, errback) => {
                try {
                    console.log(`Producing ${kind}...`);
                    const { id } = await socketRequest('produce', {
                        transportId: sendTransport.id,
                        kind,
                        rtpParameters,
                    });
                    console.log(`✅ Producer created: ${id} (${kind})`);
                    callback({ id });
                } catch (error) {
                    console.error('❌ Produce error:', error);
                    errback(error);
                }
            });
        }

        async function createRecvTransport() {
            const { id, iceParameters, iceCandidates, dtlsParameters } = await socketRequest('createTransport', { type: 'recv' });

            recvTransport = device.createRecvTransport({
                id,
                iceParameters,
                iceCandidates,
                dtlsParameters,
            });

            recvTransport.on('connect', async ({ dtlsParameters }, callback, errback) => {
                try {
                    console.log('Connecting recv transport...');
                    await socketRequest('connectTransport', {
                        transportId: recvTransport.id,
                        dtlsParameters,
                    });
                    recvTransportConnected = true;
                    recvTransport._connected = true;
                    console.log('✅ Recv transport connected, state:', recvTransport.connectionState);
                    callback();
                } catch (error) {
                    console.error('❌ Recv transport connection error:', error);
                    recvTransportConnected = false;
                    recvTransport._connected = false;
                    errback(error);
                }
            });

            recvTransport.on('connectionstatechange', (state) => {
                console.log(`📡 Recv transport connection state: ${state}`);
                if (state === 'connected' || state === 'completed') {
                    recvTransportConnected = true;
                    recvTransport._connected = true;
                } else if (state === 'failed' || state === 'disconnected') {
                    recvTransportConnected = false;
                    recvTransport._connected = false;
                    console.error('Recv transport connection failed or disconnected');
                }
            });
        }

        // Check permission state before requesting
        async function checkPermissions() {
            try {
                if (navigator.permissions && navigator.permissions.query) {
                    const cameraPermission = await navigator.permissions.query({ name: 'camera' }).catch(() => null);
                    const microphonePermission = await navigator.permissions.query({ name: 'microphone' }).catch(() => null);
                    
                    if (cameraPermission && cameraPermission.state === 'denied') {
                        return { denied: true, reason: 'Camera permission was denied. Please enable it in browser settings.' };
                    }
                    if (microphonePermission && microphonePermission.state === 'denied') {
                        return { denied: true, reason: 'Microphone permission was denied. Please enable it in browser settings.' };
                    }
                }
                return { denied: false };
            } catch (e) {
                // Permissions API not supported, continue anyway
                return { denied: false };
            }
        }

        async function produceMedia() {
            // Clean up any existing stream first (important for Edge)
            if (localStream) {
                localStream.getTracks().forEach(track => {
                    track.stop();
                });
                localStream = null;
            }

            // Wait a bit for device to be released (especially for Edge)
            await new Promise(resolve => setTimeout(resolve, 300));

            // Check permissions first
            const permissionCheck = await checkPermissions();
            if (permissionCheck.denied) {
                showError(permissionCheck.reason);
                throw new Error('Permission denied');
            }

            // Check if getUserMedia is available
            if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
                const error = new Error('getUserMedia is not supported in this browser');
                showError(error.message);
                throw error;
            }

            let stream = null;
            let lastError = null;

            // Use selected devices from dropdowns (flexible: works with one or many devices, audio-only or video-only)
            const deviceCheck = await checkDeviceAvailability();
            const audioSelect = document.getElementById('audioDeviceSelect');
            const videoSelect = document.getElementById('videoDeviceSelect');
            const selectedAudioId = audioSelect && audioSelect.value ? audioSelect.value : null;
            const selectedVideoId = videoSelect && videoSelect.value ? videoSelect.value : null;

            const audioConstraint = deviceCheck.hasAudio ? {
                ...(selectedAudioId && { deviceId: { exact: selectedAudioId } }),
                echoCancellation: true,
                noiseSuppression: true,
                autoGainControl: true
            } : false;
            const videoConstraint = deviceCheck.hasVideo ? {
                ...(selectedVideoId && { deviceId: { exact: selectedVideoId } }),
                width: { ideal: 1280, max: 1920 },
                height: { ideal: 720, max: 1080 },
                frameRate: { ideal: 30, max: 60 }
            } : false;

            // Try with ideal constraints first
            try {
                stream = await navigator.mediaDevices.getUserMedia({
                    audio: audioConstraint,
                    video: videoConstraint
                });
            } catch (error) {
                lastError = error;
                
                // If permission denied, stop immediately - don't retry
                if (error.name === 'NotAllowedError') {
                    showError('Permission denied. Please allow camera/microphone access and try again.');
                    throw error;
                }
                
                // If device not found, stop immediately
                if (error.name === 'NotFoundError') {
                    showError('No camera/microphone found. Please connect a device and try again.');
                    throw error;
                }
                
                // If overconstrained, try with basic constraints
                if (error.name === 'OverconstrainedError' || error.name === 'NotReadableError') {
                    console.warn('⚠️ Trying with basic constraints...');
                    const basicAudio = deviceCheck.hasAudio ? (selectedAudioId ? { deviceId: { exact: selectedAudioId } } : true) : false;
                    const basicVideo = deviceCheck.hasVideo ? (selectedVideoId ? { deviceId: { exact: selectedVideoId } } : true) : false;
                    try {
                        stream = await navigator.mediaDevices.getUserMedia({
                            audio: basicAudio,
                            video: basicVideo
                        });
                    } catch (fallbackError) {
                        lastError = fallbackError;
                        
                        // If still permission denied, stop
                        if (fallbackError.name === 'NotAllowedError') {
                            showError('Permission denied. Please allow camera/microphone access and try again.');
                            throw fallbackError;
                        }
                        
                        // If device in use, wait and retry once more
                        if (fallbackError.name === 'NotReadableError') {
                            console.warn('⚠️ Device in use, waiting 1 second before final attempt...');
                            await new Promise(resolve => setTimeout(resolve, 1000));
                            
                            try {
                                stream = await navigator.mediaDevices.getUserMedia({ audio: basicAudio, video: basicVideo });
                            } catch (finalError) {
                                showError('Device is in use by another application. Please close other apps using the camera/microphone and refresh the page.');
                                throw finalError;
                            }
                        } else {
                            throw fallbackError;
                        }
                    }
                } else {
                    throw error;
                }
            }

            if (!stream) {
                const error = new Error('Failed to get media stream');
                showError(error.message);
                throw error;
            }

            try {
                localStream = stream;
                addLocalVideo(stream);

                // Produce audio
                const audioTrack = stream.getAudioTracks()[0];
                if (audioTrack) {
                    const audioProducer = await sendTransport.produce({ track: audioTrack });
                    producers.set('audio', audioProducer);
                    producers.set(audioProducer.id, audioProducer);
                    console.log(`✅ Audio producer created: ${audioProducer.id}`);
                }

                // Produce video
                const videoTrack = stream.getVideoTracks()[0];
                if (videoTrack) {
                    const videoProducer = await sendTransport.produce({ track: videoTrack });
                    producers.set('video', videoProducer);
                    producers.set(videoProducer.id, videoProducer);
                    console.log(`✅ Video producer created: ${videoProducer.id}`);
                }

            } catch (error) {
                // Clean up stream if production fails
                if (stream) {
                    stream.getTracks().forEach(track => track.stop());
                    localStream = null;
                }
                console.error('Error producing media:', error);
                showError('Failed to start media production: ' + error.message);
                throw error;
            }
        }

        async function consumeProducer(producerId, kind = null) {
            try {
                if (consumers.has(producerId)) return;

                if (!recvTransport || !device || !device.loaded) {
                    throw new Error('Not ready');
                }

                const socketId = producerToSocket.get(producerId);
                if (socketId === socket.id) return;

                // Optimized: Faster retries (max 300ms instead of 2000ms)
                let retries = 0;
                while (
                    recvTransport.connectionState !== 'connected' &&
                    recvTransport.connectionState !== 'completed' &&
                    retries < 6
                ) {
                    await new Promise(resolve => setTimeout(resolve, 50));
                    retries++;
                }
                
                // Request consumer creation from server
                const { id, producerId: prodId, kind: mediaKind, rtpParameters } = await socketRequest('consume', {
                    transportId: recvTransport.id,
                    producerId,
                    rtpCapabilities: device.rtpCapabilities,
                });

                // Create consumer
                const consumer = await recvTransport.consume({
                    id,
                    producerId: prodId,
                    kind: mediaKind,
                    rtpParameters,
                });

                consumers.set(producerId, consumer);
                consumer.track.enabled = true;

                // Resume consumer (non-blocking)
                socketRequest('resumeConsumer', { consumerId: consumer.id }).catch(() => {});

                // Get track and add to video element immediately
                const { track } = consumer;
                if (!track) throw new Error('Consumer has no track');
                
                // Monitor track state (minimal)
                track.onended = () => {
                    const socketId = producerToSocket.get(producerId);
                    if (socketId) {
                        const stream = socketToStream.get(socketId);
                        if (stream) stream.removeTrack(track);
                    }
                };
                
                // Add track to video element immediately (no waiting)
                await addRemoteVideo(track, producerId, mediaKind);

            } catch (error) {
                console.error('Error consuming producer:', error);
                // Don't show error to user for every failed consume, just log it
                console.error(`Failed to consume producer ${producerId}: ${error.message}`);
            }
        }

        function buildParticipantOverlay(name, avatar) {
            const wrap = document.createElement('div');
            wrap.className = 'video-participant-info';

            const avatarWrap = document.createElement('div');
            avatarWrap.className = 'participant-avatar-wrap';

            if (avatar) {
                const img = document.createElement('img');
                img.className = 'participant-avatar';
                img.src = avatar;
                img.alt = name;
                img.style.display = 'block';
                avatarWrap.appendChild(img);
            } else {
                const fb = document.createElement('div');
                fb.className = 'participant-avatar-fallback';
                fb.textContent = (name || '?').charAt(0).toUpperCase();
                fb.style.display = 'flex';
                avatarWrap.appendChild(fb);
            }

            const nameEl = document.createElement('div');
            nameEl.className = 'participant-name';
            nameEl.textContent = name || 'Participant';

            wrap.appendChild(avatarWrap);
            wrap.appendChild(nameEl);
            return wrap;
        }

        function addLocalVideo(stream) {
            const videoGrid = document.getElementById('videoGrid');
            const videoContainer = document.createElement('div');
            videoContainer.className = 'video-container';
            videoContainer.id = 'local-video';

            const video = document.createElement('video');
            video.srcObject = stream;
            video.autoplay = true;
            video.playsInline = true;
            video.muted = true;

            const localName = (document.getElementById('username') && document.getElementById('username').value) || 'You';
            const overlay = buildParticipantOverlay(localName + ' (You)', null);

            videoContainer.appendChild(video);
            videoContainer.appendChild(overlay);
            videoGrid.appendChild(videoContainer);
        }

        async function addRemoteVideo(track, producerId, kind) {
            const socketId = producerToSocket.get(producerId);
            if (!socketId) return;

            const videoGrid = document.getElementById('videoGrid');
            let stream = socketToStream.get(socketId);
            
            if (!stream) {
                stream = new MediaStream();
                socketToStream.set(socketId, stream);
            }
            
            if (!stream.getTracks().find(t => t.id === track.id)) {
                stream.addTrack(track);
            }
            
            track.enabled = true;
            
            const pInfo = socketParticipantInfo.get(socketId);
            const pName = pInfo?.name || 'Participant';
            const pAvatar = pInfo?.avatar || null;

            // Get or create video container
            let videoContainer = document.getElementById(`remote-${socketId}`);
            let video = document.getElementById(`video-${socketId}`);
            
            if (!videoContainer) {
                videoContainer = document.createElement('div');
                videoContainer.className = 'video-container';
                videoContainer.id = `remote-${socketId}`;
                videoGrid.appendChild(videoContainer);
            }

            if (!video) {
                video = document.createElement('video');
                video.id = `video-${socketId}`;
                video.autoplay = true;
                video.playsInline = true;
                video.muted = false;
                video.style.cssText = 'width:100%;height:100%;object-fit:cover;display:block;background:#000';

                const overlay = buildParticipantOverlay(pName, pAvatar);
                overlay.id = `overlay-${socketId}`;
                
                videoContainer.appendChild(video);
                videoContainer.appendChild(overlay);
            } else {
                // Update overlay if participant info arrived after initial render
                const existingOverlay = document.getElementById(`overlay-${socketId}`);
                if (existingOverlay) existingOverlay.remove();
                const overlay = buildParticipantOverlay(pName, pAvatar);
                overlay.id = `overlay-${socketId}`;
                videoContainer.appendChild(overlay);
            }

            // Set stream immediately (no delays)
            video.srcObject = stream;
            video.play().catch(() => {}); // Non-blocking play attempt
        }

        function removeParticipant(socketId) {
            // Find all producers for this socket and remove their consumers
            const producersToRemove = [];
            for (const [producerId, sId] of producerToSocket.entries()) {
                if (sId === socketId) {
                    producersToRemove.push(producerId);
                }
            }

            // Remove consumers
            for (const producerId of producersToRemove) {
                const consumer = consumers.get(producerId);
                if (consumer) {
                    consumer.close();
                    consumers.delete(producerId);
                }
                producerToSocket.delete(producerId);
            }
            
            // Remove stream tracking
            socketToStream.delete(socketId);
            
            // Remove video element
            const videoContainer = document.getElementById(`remote-${socketId}`);
            if (videoContainer) {
                videoContainer.remove();
            }
        }

        async function toggleAudio() {
            if (localStream) {
                const audioTrack = localStream.getAudioTracks()[0];
                if (audioTrack) {
                    isAudioEnabled = !isAudioEnabled;
                    audioTrack.enabled = isAudioEnabled;
                    
                    const btn = document.getElementById('toggleAudio');
                    btn.textContent = isAudioEnabled ? '🔊 Unmute' : '🔇 Mute';
                    btn.classList.toggle('muted', !isAudioEnabled);
                }
            }
        }

        async function toggleVideo() {
            if (localStream) {
                const videoTrack = localStream.getVideoTracks()[0];
                if (videoTrack) {
                    isVideoEnabled = !isVideoEnabled;
                    videoTrack.enabled = isVideoEnabled;
                    
                    const btn = document.getElementById('toggleVideo');
                    btn.textContent = isVideoEnabled ? '📹 Video On' : '📷 Video Off';
                    btn.classList.toggle('video-off', !isVideoEnabled);

                    // Toggle local video display
                    const localVideo = document.querySelector('#local-video video');
                    if (localVideo) {
                        localVideo.style.display = isVideoEnabled ? 'block' : 'none';
                    }
                }
            }
        }

        async function leaveRoom() {
            try {
                // Stop local stream first (important for Edge)
                if (localStream) {
                    localStream.getTracks().forEach(track => {
                        track.stop();
                        track.enabled = false;
                    });
                    localStream = null;
                }
                
                // Wait a bit for device to be released (especially for Edge)
                await new Promise(resolve => setTimeout(resolve, 200));

                // Close producers
                for (const producer of producers.values()) {
                    producer.close();
                }
                producers.clear();

                // Close consumers
                for (const consumer of consumers.values()) {
                    consumer.close();
                }
                consumers.clear();
                producerToSocket.clear();
                socketToStream.clear();
                socketParticipantInfo.clear();

                // Close transports
                if (sendTransport) {
                    sendTransport.close();
                }
                if (recvTransport) {
                    recvTransport.close();
                }

                // Leave mediasoup room only (stay connected to backend)
                if (socket) {
                    socket.emit('leaveRoom');
                }

                // Reset UI
                document.getElementById('videoSection').classList.remove('active');
                document.getElementById('setupSection').style.display = 'block';
                document.getElementById('videoGrid').innerHTML = '';
                updateStatus('connected');

            } catch (error) {
                console.error('Error leaving room:', error);
            }
        }

        function socketRequest(event, data = {}) {
            return new Promise((resolve, reject) => {
                if (!socket) {
                    reject(new Error('Socket not connected'));
                    return;
                }

                socket.emit(event, data, (response) => {
                    if (response.error) {
                        reject(new Error(response.error));
                    } else {
                        resolve(response);
                    }
                });
            });
        }

        function updateStatus(status) {
            const statusEl = document.getElementById('status');
            statusEl.className = 'status ' + status;
            
            const statusText = {
                'disconnected': 'Disconnected',
                'connecting': 'Connecting...',
                'connected': 'Connected',
                'in-call': 'In Call'
            };
            
            statusEl.textContent = statusText[status] || status;
        }

        function showError(message) {
            const errorEl = document.getElementById('error');
            errorEl.textContent = message;
            errorEl.classList.add('show');
            setTimeout(() => {
                errorEl.classList.remove('show');
            }, 5000);
        }

        window.addEventListener('load', () => {
            refreshDeviceList();
        });
    </script>
</body>
</html>
```








