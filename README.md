# CodeSync---Real-Time-Collaborative-Code-Editor


Real-time collaborative code editor — multiple people editing the same file at once, with live cursors and conflict-free syncing. Think "Google Docs for code."

## Why

Pair programming and code review over a call usually means someone shares their screen and everyone else watches. CodeSync lets everyone actually type in the same file at the same time, see each other's cursors, and never overwrite each other's changes — no merge conflicts, no "wait, who's editing now?"

## How it works

Every keystroke isn't just sent as raw text to other users — that approach breaks the moment two people type at the same time. Instead:

1. **Monaco Editor** (the same editor that powers VS Code) runs in the browser and gives syntax highlighting + a proper editing API.
2. Every edit in Monaco is translated into a **Yjs** operation — Yjs is a CRDT (Conflict-free Replicated Data Type) library. Instead of sending "the text is now X," it sends small, mergeable operations that every client can apply in any order and still land on the exact same final document. No central server has to decide "whose edit wins."
3. Those operations are broadcast over **Socket.io**, which keeps a live WebSocket connection open between every client and the server so updates arrive instantly, not on a page refresh.
4. The **Node.js/Express** backend relays these updates between connected clients in the same document "room" and also tracks **presence** — who's currently in the doc and where their cursor is — so you see colored cursors moving live as others type.
5. Everything ships in a **Docker** container with a multi-stage build, so the production image only contains the compiled app, not the whole dev toolchain.

```
Browser A (Monaco) ──Yjs op──▶ Socket.io ──▶ Express/Node server ──▶ Socket.io ──▶ Browser B (Monaco)
                                                     │
                                          broadcasts to all clients
                                          in the same document room
```

## Features

- Real-time multi-user editing of the same file, no lag between keystrokes.
- **Conflict-free merging** via CRDTs — concurrent edits from different users always converge to the same result on every screen.
- **Live presence** — see collaborators' cursor positions and selections as they type.
- Syntax highlighting and editor ergonomics via Monaco (same engine as VS Code).
- Served on a single domain — frontend and WebSocket backend share an origin, avoiding CORS/cross-origin WebSocket handshake issues.

## Tech Stack

| Layer | Technology |
|---|---|
| Editor | Monaco Editor |
| Sync engine | Yjs (CRDT) |
| Real-time transport | Socket.io |
| Backend | Node.js, Express |
| Deployment | Docker (multi-stage build) |

## Project Structure

```
codesync/
├── client/
│   ├── src/
│   │   ├── editor/         # Monaco setup + y-monaco binding
│   │   └── presence/       # Cursor/selection UI
│   └── package.json
├── server/
│   ├── src/
│   │   ├── socket/         # Socket.io connection + room handling
│   │   └── rooms/          # Document room state
│   └── package.json
├── Dockerfile               # Multi-stage build
└── README.md
```

> Adjust to your actual folder layout.

## Setup

```bash
git clone <repo-url>
cd codesync

# Install both client and server dependencies
cd server && npm install
cd ../client && npm install
```

**Key packages:**
```
monaco-editor
yjs
y-monaco          # binds Yjs to Monaco's editor model
y-websocket / socket.io-client
socket.io
express
```

## Running Locally

```bash
# Start the backend (WebSocket relay)
cd server && npm run dev

# Start the frontend
cd client && npm run dev
```

Open the app in two browser tabs/windows and join the same document ID to see live sync in action.

## Docker

```dockerfile
# Build stage — installs deps, compiles/bundles
FROM node:20 AS build
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Production stage — only ships what's needed to run
FROM node:20-slim
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

```bash
docker build -t codesync .
docker run -p 3000:3000 codesync
```

## Acknowledgements

Built Jan 2026 as a personal project exploring CRDT-based real-time collaboration.
