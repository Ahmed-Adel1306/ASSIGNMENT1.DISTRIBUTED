# Go RPC Concurrent Chat

A terminal chat application built with Go's `net/rpc`. The server and every client run as separate processes. The communication model is **Server Push**: each client starts a small RPC callback listener, registers its callback address with the central server, and receives messages/notifications automatically.

## Features

- Multiple concurrent clients, each with a unique username.
- Join and leave notifications to all other connected clients.
- Broadcast messages to every other client (the sender does not receive their own message).
- `/users` lists currently connected users.
- `/quit` or Ctrl+C disconnects cleanly.
- Duplicate username rejection.
- Callback heartbeat detects unexpected client disconnections (normally within a few seconds).
- Mutex-protected shared server state and a per-client outbound event queue.
- Ctrl+C / SIGTERM graceful server shutdown.

## Requirements

- Go 1.21 or newer.
- The server and clients must be able to reach each other over TCP.

## Run locally

Open a terminal in the project folder.

### 1. Start the server

```bash
go run ./server
```

The default address is `:9000`. To use another port:

```bash
go run ./server -addr :9001
```

### 2. Start clients

Open two or more additional terminals, from the same project folder:

```bash
go run ./client
```

Enter a different username in each terminal. To connect to a server on another host/port, pass its address:

```bash
go run ./client 192.168.1.20:9000
```

**Network note:** The callback listener currently binds to `127.0.0.1`, which is ideal for multiple local terminal clients on one computer and for the requested screen-recording demonstration. For clients on different computers, change the callback listener bind address in `client/client.go` to a reachable interface and ensure the server can connect back to that client; callback addresses must be reachable from the server.

## Client commands

- Type any text and press Enter: send a message to all other clients.
- `/users`: show connected usernames.
- `/quit` or Ctrl+C: leave the chat.

## Example demonstration

1. Run one server terminal.
2. Run Client 1 with username `Ahmed`.
3. Run Client 2 with username `Mona`.
4. Run Client 3 with username `Ali`.
5. Show the join notifications in the other clients.
6. Send a message from Ahmed and show that Mona and Ali receive it, while Ahmed does not receive his own message.
7. Run `/users` in one client.
8. Run `/quit` in Mona's terminal and show the leave notification in Ahmed and Ali.
9. Press Ctrl+C in the server terminal and show the shutdown notification/disconnection.

## Build binaries

```bash
go build -o chat-server ./server
go build -o chat-client ./client
```

On Windows, Go will produce `chat-server.exe` and `chat-client.exe`.

## Repository layout

```text
go-rpc-chat/
├── client/
│   └── client.go
├── common/
│   └── types.go
├── server/
│   ├── main.go
│   └── server.go
├── go.mod
└── README.md
```

## Course-concept alignment

See [DESIGN.md](DESIGN.md) for a direct mapping between this implementation and the uploaded Distributed Systems lectures: autonomous networked processes (Lecture 1), client-server architecture and message passing (Lecture 4), push-style delivery (Lecture 4 plus the assignment callback requirement), concurrency/synchronization (Lecture 3), and application-level failure detection/recovery (Lecture 3). The Go `net/rpc` package is used because it is explicitly required by the assignment; it is an implementation choice, not a claim that the slides teach this Go API.

## Design notes

- `net/rpc` uses TCP and Go's `gob` encoding.
- The client exposes `Callback.Deliver` and `Callback.Ping` RPC methods on its callback listener.
- The server stores one callback RPC connection per session. Each session has its own goroutine and queue, so callback delivery for a client is serialized and does not block other clients' delivery workers.
- The server protects the connected-user map with a mutex.
- A failed callback/heartbeat removes that session and notifies the remaining clients.
