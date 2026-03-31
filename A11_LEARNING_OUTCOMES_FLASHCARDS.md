# A11 Learning Outcomes Flashcards (Concepts + Code Mapping)

This guide is organized by the listed A11 learning outcomes.
Each section has concept notes plus quick flashcard-style toggles.

Primary references used:
- README.md
- src/server.c
- src/client.c
- src/utility.c
- include/utility.h

---

## Outcome 1: Know the messaging protocol (packet structure, types, interactions)

### What to know
- Max message size is 1024 bytes.
- Byte 0 is message type.
- Type 0: regular message.
- Type 1: end-of-execution control message.
- Newline terminates each frame.

### Server/client interaction model
- Client sends type 0 payloads.
- Server rebroadcasts as type 0 plus sender metadata (IP + port).
- Client sends type 1 when done sending.
- Server sends final type 1 after all expected clients send type 1.

<details>
<summary>Flashcard check</summary>

Q: Why does the implementation keep a stream buffer per connection?
A: Because TCP is a byte stream; a frame may be split across reads or multiple frames may arrive in one read.

</details>

---

## Outcome 2: Know how to run client, server, and tester programs

### Core commands
- Server: ./server <port number> <# of clients>
- Client: ./client <IP address> <port number> <# of messages> <log file path>
- Server tester: ./server-tester <IP> <port> <#clients> <#messages> <test#>
- Client tester: ./client-tester <client path> <log prefix> <port> <#clients> <#messages> <test#>

### Build command flow
1. cmake -S . -B build
2. cmake --build build

<details>
<summary>Flashcard check</summary>

Q: Why should you still run with default tester log level before submit?
A: Grader uses default behavior; passing only with custom logging mode is not enough.

</details>

---

## Outcome 3: Know TCP networking call order in C

### Server side order
1. socket
2. bind
3. listen
4. accept
5. recv/send (or read/write)
6. close

### Client side order
1. socket
2. connect
3. send/recv
4. close

<details>
<summary>Flashcard check</summary>

Q: What is the most common error if you skip bind on the server?
A: The server has no local endpoint attached, so listen/accept path cannot work as intended.

</details>

---

## Outcome 4: Know how to find human-readable IP and port

### Concept
- Network metadata may be binary internally.
- Convert IP with inet_ntop.
- Convert network-order port to host-order with ntohs.

### In this implementation
- Server embeds sender IP/port in type-0 rebroadcast frame.
- Client parses those fields and prints human-readable values.

<details>
<summary>Flashcard check</summary>

Q: Why is ntohs needed before printing a received port?
A: Because network byte order may differ from host byte order.

</details>

---

## Outcome 5: Handle multiple concurrent TCP connections

### Concept
- Multiple clients can be active at once.
- Shared data must be protected to avoid races.

### In this implementation
- One reader thread per client.
- Shared client registry protected by clients_mutex.
- Shared queue protected by queue_mutex + queue_cond.

<details>
<summary>Flashcard check</summary>

Q: Why not let each reader directly broadcast immediately?
A: Concurrent direct sends can break global ordering guarantees.

</details>

---

## Outcome 6: Send and receive concurrently on TCP

### Concept
- Sender and receiver paths should run independently.

### In this implementation
- Client runs receiver thread while sender loop runs in main thread.
- Shutdown flag coordinates stop conditions.

<details>
<summary>Flashcard check</summary>

Q: Why start receiver thread before sending fuzz messages?
A: To avoid missing incoming broadcasts that may arrive early.

</details>

---

## Outcome 7: Implement all above in C with correct APIs

### C patterns used
- Pointer-safe argument checks.
- Manual memory allocation/free for queue nodes and thread args.
- Mutex/condition variable synchronization.
- Byte-oriented protocol packing/unpacking with memcpy.

### Key API families
- Sockets: socket, bind, listen, accept, connect, send, recv
- Conversion: inet_pton, inet_ntop, htons, ntohs
- Threads: pthread_create, pthread_detach/join, mutexes, cond vars
- Random/file I/O: getentropy, fopen, fprintf, fflush

<details>
<summary>Flashcard check</summary>

Q: Why is send_all a required systems pattern here?
A: A single send call may write fewer bytes than requested.

</details>

---

## Cross-Outcome Integration (Exam Perspective)

If you understand this chain, you are strong for A11 quiz questions:

1. TCP stream behavior -> need framing parser.
2. Framing + protocol layout -> correct byte-level parsing.
3. Multiple clients -> synchronization and ordering architecture.
4. Two-phase protocol -> coordinated termination conditions.
5. CLI and testers -> reproducible validation workflow.

<details>
<summary>Mini oral quiz prompt set</summary>

1. Explain why newline framing alone is not enough without stream buffering.
2. Explain why the server adds IP/port metadata and client does not.
3. Explain how the implementation enforces one global order across clients.
4. Explain what triggers final type-1 and server termination.

</details>

---

## Practical syntax references to review quickly

- man 2 socket
- man 2 bind
- man 2 listen
- man 2 accept
- man 2 connect
- man 2 send
- man 2 recv
- man 3 pthread_create
- man 3 pthread_mutex_lock
- man 3 pthread_cond_wait
- man 3 inet_ntop
- man 3 getentropy

---

## Syntax Deep-Dive Cards (C + Systems)

These cards focus on syntax patterns that commonly appear in quizzes.

<details>
<summary>Card: Why use ssize_t for recv/send return values?</summary>

- recv/send can return negative values on error.
- size_t is unsigned and cannot represent -1.
- ssize_t is the correct signed counterpart for byte counts with error signaling.

</details>

<details>
<summary>Card: Why are frame buffers often uint8_t instead of char?</summary>

- The protocol has binary fields (type byte, IP, port).
- uint8_t makes byte-level operations explicit and avoids accidental string assumptions.
- char-based code can accidentally mix text and binary semantics.

</details>

<details>
<summary>Card: Why use memcpy for IP/port fields in frames?</summary>

- IP and port are binary integers in protocol layout.
- memcpy copies exact bytes without text conversion.
- String formatting/parsing would violate the specified packet structure.

</details>

<details>
<summary>Card: Why check frame_len before reading metadata offsets?</summary>

- Defensive parsing prevents out-of-bounds reads.
- For server type-0 parsing, minimum valid frame length is:
- 1 byte type + 4 byte ip + 2 byte port + 1 byte newline = 8 bytes.

</details>

<details>
<summary>Card: Why does producer-consumer use both mutex and condition variable?</summary>

- Mutex protects queue state from data races.
- Condition variable allows consumer to sleep while queue is empty.
- Without condition variable, consumer would busy-wait and waste CPU.

</details>

<details>
<summary>Card: Why is send_all loop condition total_sent &lt; len?</summary>

- One send call may write only part of a frame.
- Loop continues until all bytes are sent or an error occurs.
- This preserves protocol framing integrity.

</details>

---

## Code-to-Learning-Outcome Mapping Cards

Use these to connect implementation details to outcomes quickly.

<details>
<summary>Card: Outcome check for protocol knowledge</summary>

Look at utility builder/parser functions:
- build_client_type0
- build_client_type1
- build_server_type0
- parse_server_type0

If you can explain those layouts from memory, you likely satisfy packet-structure learning goals.

</details>

<details>
<summary>Card: Outcome check for concurrent TCP handling</summary>

Look at server/client concurrency split:
- server: reader threads + broadcaster thread
- client: sender path + receiver thread

If you can explain why these run concurrently and what each protects with locks, you likely satisfy concurrency outcome goals.

</details>

<details>
<summary>Card: Outcome check for networking call order</summary>

Server startup pattern:
- socket -> bind -> listen -> accept

Client startup pattern:
- socket -> connect

If you can justify this order and common failure points, you satisfy lifecycle outcome goals.

</details>

---

## Quiz Context: How Questions Are Usually Framed

These are common quiz styles and how to approach them.

<details>
<summary>Style 1: "Spot the protocol bug"</summary>

Checklist:
1. Is type byte first?
2. Is newline delimiter present?
3. Is server type-0 carrying binary ip+port in correct order?
4. Are size checks done before memcpy?

</details>

<details>
<summary>Style 2: "What happens if recv returns 0?"</summary>

- recv == 0 means peer closed connection gracefully.
- Connection-specific cleanup should happen.
- It is not the same as partial frame success.

</details>

<details>
<summary>Style 3: "Which lock protects what?"</summary>

- clients_mutex: active client registry
- queue_mutex/queue_cond: message queue state and wake-up
- state_mutex: global termination counters and flags

</details>

<details>
<summary>Style 4: "Why same global order?"</summary>

- One broadcaster consumes one FIFO queue.
- Every client receives in broadcaster emission order.
- Avoids race-driven reorder from multiple sending threads.

</details>

---

## Rapid Recall Mini-Deck (10 one-liners)

<details>
<summary>1. First byte of every protocol frame?</summary>

Type byte (0 or 1).

</details>

<details>
<summary>2. Frame boundary marker?</summary>

Newline character.

</details>

<details>
<summary>3. Why stream buffer?</summary>

TCP is byte stream, not message stream.

</details>

<details>
<summary>4. Server type-0 payload prefix?</summary>

Binary sender IP then binary sender port.

</details>

<details>
<summary>5. Why send_all?</summary>

send can be short.

</details>

<details>
<summary>6. Client completion signal?</summary>

One type-1 message after all type-0 sends.

</details>

<details>
<summary>7. Server final stop condition?</summary>

All expected clients have sent type-1.

</details>

<details>
<summary>8. Function to stringify IPv4?</summary>

inet_ntop.

</details>

<details>
<summary>9. Function to host-convert received port?</summary>

ntohs.

</details>

<details>
<summary>10. Why dedicated broadcaster thread?</summary>

To preserve one global delivery order.

</details>
