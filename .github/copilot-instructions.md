# CS 457 Networking Game Project Instructions

## Project overview
- This repository contains a Python 3 console Battleship/networking project. The application consists of `client.py` and `server.py`.
- The runnable source files are currently simplified socket-selector examples to use as a base for the program. Treat the Markdown specifications in `CS457-Networking-Game-Project/` as the intended implementation requirements unless the user explicitly approves a change.
- Preserve the educational/course nature of the code: prefer clear, modular Python over unnecessary frameworks or abstractions.

## Source-of-truth hierarchy
1. `ai-instruction-sheet.md` is the Sprint 3 grading and implementation checklist.
2. `protocol_blueprint.md` defines the transport, framing, message types, and JSON packet shape.
3. `fsm_documentation.md` and `sow_template.md` define the state transitions and Battleship behavior.
4. `concurrency_design.md` defines the intended non-blocking `selectors` server model and two-client limit.
5. `sequence_diagram.md` and `concurrency_design.md` clarify turn, error, and disconnect behavior.
6. Existing Python source is scaffolding and is not authoritative over the specifications.

When specifications conflict, do not silently choose a different contract. Call out the conflict and ask before making a behavior-changing implementation.

## Network and protocol requirements
- Use Python 3, IPv4 TCP sockets, `socket.AF_INET`, and `socket.SOCK_STREAM`.
- Client CLI:
  - `python3 client.py -h`
  - `python3 client.py -i <server-hostname> -p <port>`
  - Both `-i/--hostname` and `-p/--port` are required for normal operation.
- Server CLI:
  - `python3 server.py -h`
  - `python3 server.py -p <port>`
  - Bind to `0.0.0.0`, never to `127.0.0.1` for the listening socket.
- The client must resolve hostnames with `socket.gethostbyname()` or an equivalent explicit DNS API.
- Catch `socket.gaierror` in the client and print a clear, actionable DNS error. Exit non-zero with status `1`; do not print a raw traceback.
- Also handle connection and socket errors gracefully at application boundaries. Do not use broad exception handlers as silent success/failure fallbacks.

## Framing and JSON contract
- Use newline-delimited UTF-8 JSON (NDJSON): one JSON object per `\n`-terminated line.
- Never assume a `recv()` call contains exactly one complete message or no partial message.
- Maintain a per-socket incoming byte buffer and split complete lines only on `b"\n"`.
- Decode and `json.loads()` each complete line explicitly and report malformed JSON as an application `ERROR` or a clear local error.
- Serialize packets with `json.dumps(...).encode("utf-8")` plus `b"\n"` and preserve outgoing data in the per-socket buffer for partial `send()` calls.
- Prefer the message field names already defined in `protocol_blueprint.md`, including `packet_type`, `player_id`, `contents`, and `timestamp`. Do not rename or omit fields without updating the specification and both sides.
- Implement packet validation for required fields and packet types before using application state.

## Server architecture and game state
- Keep the server single-process and non-blocking with `selectors.DefaultSelector()`.
- Accept at most two active game players. If a third connection attempts to join, send an `ERROR` or reject it cleanly.
- Maintain the authoritative board and game state on the server. The server determines the active player, validates moves, and broadcasts state changes.
- Represent state transitions explicitly instead of relying on ad hoc socket or boolean behavior. Keep the FSM aligned with `fsm_documentation.md`.
- Enforce message types by current state:
  - only the second connected player and the number-selection phase accept `PLAYER_NUMBER`;
  - only placement state accepts valid `PLACE_SHIP` packets;
  - only the active player in gameplay accepts `PLAYER_TURN`;
  - `DISCONNECT` is valid outside turn rules and must be handled safely.
- Send `ERROR` for malformed packets, out-of-state packet types, duplicate guesses, invalid coordinates, and moves from a player who is not active.
- Before removing or resetting state, unregister and close sockets safely and preserve the remaining player's disconnect/game-over behavior.

## Client architecture
- Keep terminal output human-readable and keep CLI prompts distinct from protocol traffic.
- Render board and turn state in the CLI after server updates; do not trust client-side state for move or win authority.
- Handle server-initiated `CONN_WAIT`, `CONN_CONFIRM`, `GAME_START`, `SHIPS_PLACED`, `TURN_RESULT`, `GAME_OVER`, and `ERROR` packets according to the protocol.
- Treat partial reads, writes, clean EOF, `BrokenPipeError`, and `ConnectionResetError` as normal network cases with clean logging and exit behavior.

## Code style and structure
- Use PEP 8 formatting and 4-space indentation.
- Use descriptive snake_case names and type hints for new public functions and protocol/state structures where practical.
- Keep networking I/O, protocol framing/serialization, FSM logic, and Battleship board logic separate enough to test independently.
- Add concise comments only for non-obvious protocol, framing, state-transition, or concurrency behavior.
- Avoid changing both `client.py` and `server.py` to equivalent echo behavior: implementation work must follow the Battleship protocol.
- Do not add external dependencies unless the task cannot reasonably be completed with Python's standard library.

## Validation
- Before changing code, inspect all relevant files: the Python entry points and the four design documents that govern the affected behavior.
- Run the smallest relevant syntax check first:
  - `python3 -m py_compile client.py server.py`
- Run or add focused tests for introduced behavior, especially NDJSON framing, DNS failure handling, CLI parsing, duplicate/out-of-turn packets, partial socket sends/reads, and disconnect cleanup.
- For local integration validation, run the server first, then start two clients. Exercise connection, number guessing, ship placement, at least three turns, invalid moves, and game-over/disconnect paths.
- Report the exact commands and their outcomes. Do not claim a network or multiplayer test passed unless it was actually run.
