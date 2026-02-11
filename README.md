## Project Overview

This repository implements a **real-time multiplayer chess game** using Node.js, Express, Socket.IO, and chess.js.  
It enables two players to play against each other in a shared browser-based chessboard, with additional clients connected as spectators.

The primary purposes inferred from the code are:

- **Demonstrate server-authoritative chess logic** using chess.js on the backend.
- **Showcase real-time bidirectional communication** with Socket.IO for broadcasting moves and synchronizing board state.
- **Provide a minimal but functional drag-and-drop chess UI** rendered in the browser without external board libraries.

There is no authentication, persistence, matchmaking, or multi-game support. The application is intentionally simple, focusing on a **single shared game instance** per server process.

---

## Tech Stack

### Languages

- **JavaScript (Node.js)** – server-side logic (`app.js`).
- **JavaScript (browser)** – client-side game and UI logic (`public/js/chessgame.js`).
- **HTML + EJS** – rendering the main page (`views/index.ejs`).
- **CSS** – Tailwind classes (via CDN) plus inline styles in `index.ejs`.

### Frameworks & Libraries

- **Express** (`express`):
  - HTTP server and routing.
  - View rendering with EJS.
- **Socket.IO** (`socket.io`):
  - Real-time communication between clients and server.
- **EJS** (`ejs`):
  - View template engine for the main index page.
- **chess.js**:
  - **Server:** npm dependency `^1.4.0` (imported as `Chess` in `app.js`).
  - **Client:** CDN script `0.10.3` in `index.ejs`.
- **Tailwind CSS** (CDN via `@tailwindcss/browser@4`):
  - Utility classes for layout and styling of the main page.

### Tooling

- **Node.js runtime**:
  - Project configured as `"type": "commonjs"` in `package.json`.
- **npm**:
  - Dependency management via `package.json` and `package-lock.json`.
  - Single built-in script: `npm test` (stub only, no real tests).

---

## High-Level Architecture

### Overview

The system consists of **three primary layers**:

1. **HTTP Layer (Express)**
   - Serves a single route `GET /` that renders the chess page via EJS.
   - Serves static assets (JavaScript) from the `public` directory.

2. **Real-Time Game Layer (Socket.IO + chess.js on the server)**
   - Maintains a **single `Chess` instance** representing the current game state.
   - Manages **player roles** (white, black, spectator) based on socket connections.
   - Validates and applies moves server-side, then broadcasts updates to all clients.

3. **Client UI Layer (Browser JavaScript)**
   - Renders the board using the chess.js board representation.
   - Implements drag-and-drop for moving pieces.
   - Connects via Socket.IO to receive role assignments and game state updates.
   - Updates its local chess.js instance based on server events.

### Data Flow

**Connection and Role Assignment**

1. Client requests `GET /`.
2. Express renders `views/index.ejs`.
3. The page loads:
   - Tailwind CSS (CDN).
   - Socket.IO client (CDN).
   - chess.js (client version, CDN).
   - `public/js/chessgame.js`.
4. Client establishes a Socket.IO connection to the server.

On each new connection (`io.on("connection", ...)` in `app.js`):

- If `players.white` is empty → this socket becomes **white** (`"w"`).
- Else if `players.black` is empty → this socket becomes **black** (`"b"`).
- Else → this socket is a **spectator**.

Role is communicated via events:

- `playerRole` (payload `"w"` or `"b"`).
- `spectatorRole` (no payload, indicates spectator).

**Move Flow**

- Player drags a piece on the client:
  - Client computes a move object `{ from, to, promotion: 'q' }` in algebraic notation (e.g., `"e2" → "e4"`).
  - Client emits `socket.emit("move", move)`.

On the server (`uniquesocket.on("move", ...)` in `app.js`):

- Checks that the socket corresponds to the correct side to move:
  - White’s turn ⇒ only `players.white` may move.
  - Black’s turn ⇒ only `players.black` may move.
- Attempts to apply move via `chess.move(move)`:
  - If valid:
    - Updates internal game state.
    - Broadcasts `move` event to all sockets.
    - Broadcasts the current FEN via `boardState` to all sockets.
  - If invalid:
    - Logs error.
    - Emits `invalidMove` back to the initiating socket.

On the client:

- `move` event → locally apply `chess.move(move)` and re-render.
- `boardState` event → `chess.load(fen)` and re-render (synchronizes full board).
- Board orientation is adjusted:
  - White: normal orientation.
  - Black: board and pieces rotated 180° (`.flipped` class).

**Disconnection Handling**

- On `disconnect`:
  - If the socket ID matches `players.white` or `players.black`, that entry is deleted.
  - No automatic re-assignment is done; new connections can occupy freed roles.

---

## Folder Structure

```text
chess-game/
├── app.js
├── package.json
├── package-lock.json
├── README.md
├── .gitignore
├── public/
│   └── js/
│       └── chessgame.js
└── views/
    └── index.ejs
```

### Root Level

- **`app.js`**  
  Main server entrypoint. Configures Express, Socket.IO, the chess game state, role assignment, and move handling.

- **`package.json`**  
  Node.js package metadata and npm dependencies (`express`, `socket.io`, `ejs`, `chess.js`). Defines `"main": "app.js"`, but only a test script is provided.

- **`package-lock.json`**  
  Auto-generated dependency lockfile pinning specific versions and dependency tree.

- **`README.md`**  
  Project documentation (this file).

- **`.gitignore`**  
  Standard Node.js ignore rules (e.g., `node_modules/`). Also mentions `.env`, although none is actually used in the code.

### `public/` – Static Assets

- **`public/js/chessgame.js`**  
  Client-side script:
  - Connects to Socket.IO.
  - Uses chess.js (browser version) to track board state.
  - Renders an 8×8 grid into a `.chessboard` container.
  - Implements drag-and-drop behavior for movable pieces.
  - Listens for role and game state events from the server.

Express is configured to serve this directory statically:

```js
app.set('view engine', "ejs")
app.use(express.static("public"));
```

### `views/` – Templates

- **`views/index.ejs`**  
  The only view template:
  - Defines the HTML `head` and `body`.
  - Includes Tailwind CSS via CDN.
  - Defines inline styles for `.chessboard`, `.square`, `.piece`, and `.flipped`.
  - Contains the main board container (`<div class="chessboard ..."></div>`).
  - Includes client-side scripts: Socket.IO client, chess.js (CDN), and `/js/chessgame.js`.

---

## Core Modules & Features

### Server Module: `app.js`

**Responsibilities**

- Initialize Express app and HTTP server.
- Create and manage the `Chess` game instance.
- Track which socket IDs map to white and black players.
- Enforce turn-based moves using server-side chess.js.
- Broadcast game state to all connected clients via Socket.IO.

**Core State**

```js
const chess = new Chess()

let players = {}
let currentPlayer = 'W'
```

### Client Module: `public/js/chessgame.js`

**Responsibilities**

- Maintain a client-side chess.js representation of the board.
- Render the current board into the DOM.
- Handle drag-and-drop interactions for legal pieces (by role).
- Communicate with the server via Socket.IO.

**Move Construction**

```js
const handleMove = (source, target) => {
  const move = {
    from : `${String.fromCharCode(97+source.col)}${8-source.row}`,
    to : `${String.fromCharCode(97+target.col)}${8-target.row}`,
    promotion: 'q'
  }

  socket.emit("move", move)
}
```

### View Module: `views/index.ejs`

**Responsibilities**

- Provide the HTML shell and CSS needed for the chessboard.
- Load external dependencies from CDNs.
- Place a `.chessboard` container in the center of a dark background.

---

## Database Models & Schemas

There is **no database or persistence layer** implemented in this repository.

- No use of ORMs (e.g., Mongoose, Sequelize) or raw SQL.
- No configuration for external datastores (e.g., MongoDB, PostgreSQL, Redis).
- Game-related state is strictly **in-memory**, held in:

  - `chess` – a single `Chess` instance.
  - `players` – object mapping roles to current socket IDs.

All state is lost when:

- The server process stops or restarts.
- The process crashes.

There is no persistence of game history, user accounts, or statistics.

---

## API Endpoints / Routes

### HTTP Routes (Express)

- **GET `/`**  
  Serves and renders the main chess game page (`index.ejs`).

### WebSocket / Socket.IO Events

#### Client → Server

- **`move`**  
  **Payload**:

  ```js
  {
    from: "e2",
    to: "e4",
    promotion: "q"
  }
  ```

  **Semantics**:

  - Expresses a requested chess move from the current client.
  - Must be legal according to chess.js and consistent with whose turn it is.

#### Server → Client

- **`playerRole`** – assigns `"w"` or `"b"` to the client.
- **`spectatorRole`** – marks the client as a spectator.
- **`move`** – broadcast of an accepted move, to be applied locally by clients.
- **`boardState`** – FEN string representing the entire board state for resync.
- **`invalidMove`** – indicates a rejected move (currently not handled on client).

---

## Business Logic & Workflows

### Player Role Assignment Workflow

1. New client connects to Socket.IO.
2. Server assigns:
   - First connection → white.
   - Second connection → black.
   - All subsequent connections → spectators.
3. On disconnect:
   - If a player disconnects, their slot (`white` or `black`) is freed for future connections.

### Move Processing Workflow

1. Client constructs a move via drag-and-drop and emits `move`.
2. Server:
   - Verifies the socket belongs to the side whose turn it is.
   - Uses `chess.move(move)` to validate and apply.
3. If valid, server broadcasts `move` and `boardState` to all clients.
4. Clients update their local chess.js instance and re-render the board.

---

## Configuration & Environment Variables

### Configuration

- **Port**:
  - Hardcoded to `3000`:

    ```js
    server.listen(3000, ()=>{
      console.log("Server running on port: 3000")
    })
    ```

- **Static directory**:
  - `"public"` is used as the static assets directory (`app.use(express.static("public"));`).

- **View engine**:
  - Configured as `"ejs"` via `app.set('view engine', "ejs")`.

### Environment Variables

- No environment variables are referenced anywhere in the codebase.
- `.gitignore` includes `.env`, but no `.env` file is present and none is used.

---

## Setup & Installation

### Prerequisites

- **Node.js** (recent LTS version).
- **npm** (bundled with Node.js).

### Installation Steps

```bash
git clone <repository-url>
cd chess-game
npm install
```

> Note: Replace `<repository-url>` with your actual GitHub repository URL.

---

## How to Run the Project

### Development

Run the server directly with Node:

```bash
node app.js
```

Then open:

```text
http://localhost:3000/
```

- First browser window → white.
- Second browser window → black.
- Additional windows → spectators.

### Production (Basic)

For a simple production run:

```bash
NODE_ENV=production node app.js
```

> `NODE_ENV` is not used in the code, but this is a common convention for deployments. For real production usage, add a process manager (PM2, etc.) and reverse proxy (Nginx, etc.)—these are **not** included in this repository.

---

## Assumptions & Design Decisions

- **Single Game Per Server**
  - One global `Chess` instance and a simple `players` object.

- **Ephemeral, Anonymous Users**
  - No authentication or user identification beyond transient `socket.id`.

- **Server-Authoritative Game State**
  - All moves validated by the server using chess.js.

- **Minimal UI / UX**
  - No move list, timers, or detailed game end states in the UI.

- **Simplified Promotion**
  - Promotion always to queen (`promotion: 'q'`).

- **Orientation Based on Role**
  - Board rotated for black using a `.flipped` CSS class.

- **No Persistence**
  - All game state is in-memory and lost on restart.

---

## Limitations & Known Issues

- **No Automated Tests**
  - `npm test` is a stub and always exits with an error.

- **No Multi-Game or Room Support**
  - Only a single global game instance.

- **Lack of Authentication and Identity**
  - No user accounts or security model.

- **No Reconnection Logic**
  - Disconnecting and reconnecting does not restore your prior role.

- **Incomplete End-Game UX**
  - No explicit UI for checkmate, stalemate, or resignation.

- **Promotion Hardcoded to Queen**
  - Underpromotion is not supported.

- **Redundant State Updates**
  - Both `move` and `boardState` are broadcast on each move.

- **Hardcoded Port**
  - Port configuration not driven by environment variables.

- **Client Ignores `invalidMove`**
  - Server emits it, but client does not handle or show feedback.

---

## Future Improvements

- Add a **configuration layer** (`process.env.PORT`, etc.).
- Improve UX with:
  - Turn indicators.
  - Role indicators (white, black, spectator).
  - Game status messages and move history.
- Implement a **promotion UI** for choosing piece type.
- Support **multiple games** using Socket.IO rooms and per-game `Chess` instances.
- Add a **persistence layer** (database) for storing games.
- Introduce a **test suite** (e.g., Jest) and real `npm test` commands.
- Enhance **error handling and logging** on both client and server.
- Align **chess.js versions** between client and server.

---

## Conclusion

This repository provides a concise, functional example of a **real-time, server-authoritative multiplayer chess game** using Node.js, Express, Socket.IO, and chess.js. It is ideal as a learning resource, a foundation for more advanced features, or a portfolio piece demonstrating competence in full-stack JavaScript, real-time communication, and basic game logic.

While not production-ready in its current form, the architecture is clean and easily extensible, making it a solid starting point for building a more complete online chess platform or other real-time board game applications.
