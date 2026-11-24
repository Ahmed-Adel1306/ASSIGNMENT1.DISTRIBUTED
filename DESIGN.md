# Design alignment with Distributed Systems lectures

This project implements the assignment using the terminology and concepts covered in the uploaded Distributed Systems lectures. The assignment specifically requires Go `net/rpc`; that library choice is an implementation constraint, while the architectural reasoning below follows the course material.

## 1. Distributed-system view (Lecture 1)

The application has multiple autonomous processes (one server process and multiple client processes) connected through a TCP network. They cooperate to provide one chat service. The central server coordinates the shared chat service and the clients use it to exchange messages.

## 2. Client-server architecture (Lecture 4)

The implementation follows the lecture's Client/Server Model:

- **Server:** a service process that accepts requests from clients and responds to them.
- **Client:** a process that invokes server operations through its client interface.
- **Requests and responses:** exchanged as messages.

The server exposes `Chat.Join`, `Chat.SendMessage`, `Chat.ListUsers`, and `Chat.Leave` RPC methods. Clients invoke these methods using Go `net/rpc`.

## 3. Message passing (Lecture 4)

The lecture explains that distributed processes do not share a common memory space and communicate by passing messages. This project therefore keeps the server's user/session state on the server and transfers request/event data over RPC; clients do not directly access the server's memory.

## 4. Push-style delivery (Lecture 4 + assignment requirement)

The lecture introduces a push model as server-initiated delivery, in contrast to a pull model where the client repeatedly initiates requests. The assignment asks for Server Push using an RPC callback. Each client therefore runs a small callback RPC listener and registers its reachable callback address with the server. The server calls `Callback.Deliver` to push chat messages and join/leave notifications without client polling.

The lecture's push-model example is conceptual; the RPC callback mechanism is the concrete implementation selected to satisfy this assignment.

## 5. Concurrency and synchronization (Lecture 3)

Multiple clients may invoke operations at the same time. The server uses goroutines to serve concurrent RPC connections and a mutex to protect the shared map of connected users. Each client session has its own outbound event queue and delivery goroutine. This serializes events for an individual client while allowing different clients to receive events concurrently.

The mutex-protected updates prevent unsafe concurrent modifications to the shared user list, addressing the lecture's warning that concurrent access must be synchronized to avoid concurrency problems such as lost updates.

## 6. Fault tolerance and availability (Lecture 3)

The server detects failed callback connections through RPC delivery failures and periodic heartbeat calls. A failed session is removed and the remaining clients receive a leave notification. This is a small application-level recovery mechanism; it is not a claim of full fault tolerance or replicated-server availability.

## 7. Scope and limitations

- The implementation uses one central server, as required by the assignment's client-server architecture.
- The included callback listener binds to loopback for a same-computer, multi-terminal demonstration. For clients on separate computers, bind the callback listener to a reachable network interface and configure routing/firewall access so the server can connect back to each client.
- Go `net/rpc` over TCP is used because the assignment explicitly requires it. The lecture material motivates message passing and client-server communication but does not prescribe this Go package.
