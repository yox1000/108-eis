# Real‑Time Collaborative Whiteboard

A production‑grade, low‑latency, multi‑user drawing board built with CRDT sync and offline support. This README is the single source of truth for scope, architecture, and execution.

## 1) Scope

P0 must‑haves

* Freehand pen and eraser with color and size
* Live cursors with user names and colors
* Realtime sync across multiple users and tabs
* Offline edits that merge on reconnect
* Undo and redo per user for strokes they created
* Join by room link with a signed token
* Server persistence across restarts
* One‑step container deploy

P1 if time remains

* Shapes and text labels
* Zoom and pan with viewport constraints
* Export current board as PNG and SVG
* Read‑only guest mode per room
* Snapshot compaction to shrink storage

P2 stretch

* Multi‑page boards
* Replay timeline
* Peer to peer transport as an option

Non‑goals

* Accounts and user management
* Multi‑tenant billing
* Mobile app

## 2) User experience

Flows

* Create room, get a shareable link, and join
* See others joining with cursor presence
* Draw, erase, change tool, change color, change size
* Undo or redo last local stroke
* Reconnect after network loss with automatic merge

UX details

* Toolbar is small and fixed in the top left
* Cursor label shows name and color for each participant
* Board resizes to viewport and keeps pixel ratio consistent
* Eraser behaves as true erase that reveals the background

Accessibility

* Keyboard toggles for pen and eraser
* Minimum color contrast for UI controls
* Cursor labels readable on light and dark backgrounds

Browser support

* Latest Chrome, Edge, Firefox, Safari desktop

## 3) System architecture

Components

* Client application: React with Canvas 2D rendering
* Sync server: WebSocket server for CRDT documents
* Persistence: durable storage of CRDT update logs in a file‑backed database
* Reverse proxy (optional): TLS termination and health route

Data flow

* Client batches pointer samples into a stroke object and commits at stroke end
* Server receives CRDT updates and persists them
* Presence is distributed out of band and is not persisted
* New clients replay server history to current state

Availability and recovery

* Server restarts do not lose data
* Clients that reconnect apply missed updates and merge conflict‑free

## 4) Data model (room document)

Collections

* Strokes list: append‑only collection of stroke objects
* Per‑client awareness state: transient map for presence and cursor

Stroke fields

* id unique identifier
* userId opaque client identifier
* color hex string
* size brush width in pixels
* tool pen or eraser
* points flat array of x and y pairs in canvas coordinate space
* t0 stroke start timestamp in milliseconds

Presence fields

* userId
* name
* color
* cursor position x and y in canvas space
* selected tool

Undo or redo

* Client records identifiers of strokes it authored in order
* Undo removes the last authored stroke from the shared list
* Redo re‑adds a saved copy to the shared list

## 5) Sync and presence

* One CRDT document per room id
* Awareness presence is separate from the document
* Presence updates are small JSON payloads that can broadcast at high frequency
* Drawing data is committed as transactions at stroke end to reduce chatter

Offline behavior

* Client keeps a local copy of the document
* On reconnect the CRDT merges with server state and other peers without conflicts

## 6) Security and auth

* Signed token contains room id and user id with a short expiration
* Server authenticates clients per room at connect time
* Read‑write for all authenticated clients in P0
* Optional owner role in P1 for read‑only guests

Threat model notes

* Tokens are scoped to a room and are not reusable across rooms
* Presence state carries no secrets
* Transport uses TLS behind a reverse proxy in production

## 7) Persistence and recovery

* Server persists CRDT updates to a local database file
* Startup loads stored updates and serves them to clients
* Snapshot job compacts historical updates into a baseline in P1

Durability targets

* Data loss on server crash is acceptable only within the last few milliseconds before an fsync
* A board must be recoverable by at least one client reconnecting

## 8) Performance targets

Latency

* Local draw to remote paint under 150 ms on same region
* Local draw to remote paint under 300 ms cross region

Frame rate

* 60 fps during active drawing on modern laptops

Resource usage

* Memory for a 30 minute session with five users under 300 MB server side
* Document payload per stroke under 4 KB median

Tactics

* Sample pointer input with a minimum distance to avoid dense points
* Render on requestAnimationFrame instead of per network event
* Use canvas compositing for eraser to avoid reprocessing paths
* Batch CRDT updates by committing at stroke end

## 9) Repo layout

* apps/client: React app
* apps/server: WebSocket sync server
* infra: container and proxy configuration

## 10) Configuration

* JWT\_SECRET secret for signing tokens
* SQLITE\_PATH file path for server persistence for local setups
* WS\_URL client environment value with the WebSocket endpoint

Environments

* Development uses a local database file and non‑TLS WebSocket
* Production uses TLS via a reverse proxy and persistent volume for the database

## 11) Local development workflow

* Install a recent Node LTS on all machines
* Install dependencies in client and server workspaces
* Start the server in watch mode
* Start the client dev server
* Open two browser tabs to the same room with two different tokens

## 12) Deployment

* Build the server image and run it with a persistent volume for data
* Place a reverse proxy in front for TLS and health checks
* Host the client as static assets on any CDN or object storage
* Point the client to the public WebSocket URL

## 13) Testing

Manual checks

* Two tabs in the same room see each other’s cursors
* Draw and erase in one tab shows up in the other quickly
* Undo and redo only affect the strokes authored by that user
* Browser offline mode allows drawing and merges on reconnect
* Page reload restores the board state

E2E checks

* Multi‑user drawing renders the same total stroke count across clients
* Reconnection after 3 seconds of link loss maintains continuity without duplicated strokes
* Large stroke test stays above 50 fps while drawing

Load checks

* Five concurrent users drawing for two minutes does not drop messages
* Server memory growth remains bounded over repeated sessions

## 14) Observability and operations

* Structured server logs include room id and client counts on connect and disconnect
* Health route returns up status and
