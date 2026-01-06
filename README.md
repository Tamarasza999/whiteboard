#Real-Time Global Whiteboard
  [https://whiteboard.tamaraszabo.workers.dev](https://whiteboard.tamaraszabo.workers.dev)  
*Open in two browsers/devices to see real-time sync*

> A collaborative drawing board with **instant global synchronization** using Cloudflare's edge network. Experience real-time collaboration across continents with <100ms latency.

## Architecture Concepts

- ***Edge Computing:*** The WebSocket server logic runs in **300+ Cloudflare data centers** worldwide, not a central server
```java
// src/index.ts - Runs in user's nearest data center
export default {
  async fetch(request, env, ctx) {
    // Executes in 1 of 300+ edge locations worldwide
    const country = request.headers.get('CF-IPCountry');
    return handleRequest(request, env);
  }
};
```
- ***Global State Management:*** One stateful instance with strong consistency, each whiteboard room is a **Durable Object**
```java
// src/index.ts - Routing to single state instance
const id = env.WHITEBOARD_ROOM.idFromName(roomId); // Unique ID per room
const obj = env.WHITEBOARD_ROOM.get(id); // Gets the single global instance
return obj.fetch(request); // All users connect to same object
```
- ***Real-Time Protocols:*** Full-stack **WebSocket implementation** with persistent connections and JSON messaging
```java
// src/WhiteboardRoom.js
// 1. Accept WebSocket upgrade
const webSocketPair = new WebSocketPair();
webSocketPair[1].accept();

// 2. Handle messages
webSocket.addEventListener('message', (event) => {
  const data = JSON.parse(event.data); // Parse JSON
  this.broadcast(event.data, webSocket); // Send to all users
});
```
- ***Zero-Trust Security:***  All WebSocket connections **validated**, rooms **isolated**, messages **sanitized**
```java
// src/WhiteboardRoom.js 
// Validate WebSocket upgrade
if (request.headers.get('Upgrade') !== 'websocket') {
  return new Response('Expected WebSocket', { status: 426 });
}

// Validate message format
webSocket.addEventListener('message', (event) => {
  const data = JSON.parse(event.data); // Throws if invalid JSON
  // Process only valid messages
});
```


## Deployment

***1. Create Project***

***2. Configure Durable Objects***\
Configure wrangler.jsonc with SQLite migration
```javascript
"durable_objects": {
  "bindings": [{
    "name": "WHITEBOARD_ROOM",
    "class_name": "WhiteboardRoom"
  }]
}
```
***3. Build whiteboard logic***
Durable Object + frontend

##  Core Implementation

***Durable Object (Global State Manager)***
```javascript
// src/WhiteboardRoom.js
export class WhiteboardRoom {
  constructor(state, env) {
    this.sessions = []; // all connected users
    this.drawingHistory = []; // every line ever drawn
  }

  async fetch(request) {
    // handle WebSocket upgrade
    const webSocketPair = new WebSocketPair();
    this.handleSession(webSocketPair[1]);
    
    return new Response(null, {
      status: 101,
      webSocket: webSocketPair[0],
    });
  }

  broadcast(message, sender) {
    // send to ALL connected users globally
    this.sessions.forEach(session => {
      if (session !== sender) session.send(message);
    });
  }
}
```
***WebSocket Handler***

```javascript
// src/index.ts
if (url.pathname.startsWith('/room/')) {
  const roomId = url.pathname.split('/')[2];
  const id = env.WHITEBOARD_ROOM.idFromName(roomId);
  const roomObject = env.WHITEBOARD_ROOM.get(id);
  return roomObject.fetch(request); // handles real WebSocket upgrade
}