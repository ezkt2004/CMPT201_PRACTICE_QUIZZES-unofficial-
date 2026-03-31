# A11 Implementation Recap and Code Walkthrough

This recap explains what was implemented in the current A11 code, why each part exists, and how it maps to the assignment README.

Use this with:
- README.md
- include/utility.h
- src/utility.c
- src/server.c
- src/client.c
- lab9/client.c
- lab9/server_sol.c

---

## 1) Big Picture: What We Built

We moved from Lab 9 style socket examples to an A11 protocol-driven design:

1. Shared protocol helpers in utility files.
2. Multi-client server with:
- per-client reader threads,
- one global queue,
- one broadcaster thread for consistent message order.
3. Fuzzing client with concurrent send/receive and logging.
4. Two-phase termination:
- clients send type 1 when done,
- server sends final type 1 after all expected clients are done.

<details>
<summary>Why this architecture fits README requirements</summary>

- The README requires same global message order for all clients.
- A single broadcaster thread that sends messages from one FIFO queue is a clean way to enforce that order.
- The README requires concurrent send/receive on clients.
- A receiver thread plus sender loop satisfies that cleanly.

</details>

---

## 2) Build and Project Structure

### Files added/used
- CMakeLists.txt
- include/utility.h
- src/utility.c
- src/server.c
- src/client.c

### Build intent
- CMake builds exactly two executables: server and client.
- This directly matches README grading constraints.

<details>
<summary>Why include/utility.h matters</summary>

It gives one source of truth for constants and function prototypes used by both server and client.
That reduces mismatch bugs in frame formats and helper signatures.

</details>

---

## 3) Protocol Contract Implemented

README protocol essentials reflected in code:

1. Message begins with one-byte type.
- type 0: regular chat data
- type 1: completion signal

2. Messages are newline-delimited.
- '\n' marks frame end.

3. Server outbound type 0 includes sender metadata:
- [type][ip(4)][port(2)][payload][\n]

4. Max message size constrained to 1024.

<details>
<summary>Important subtlety</summary>

IP/port metadata are binary bytes, not ASCII strings. That is why parsing logic is byte-oriented and not line-splitting text only.

</details>

---

## 4) include/utility.h Overview

### Key constants
- MAX_MESSAGE_SIZE = 1024
- STREAM_BUFFER_CAP = 4096
- MAX_CLIENTS = 128
- RANDOM_BYTES_PER_MSG = 16

### Key APIs
- send_all(...)
- stream_buffer_append(...)
- stream_buffer_pop_frame(...)
- build_client_type0(...)
- build_client_type1(...)
- build_server_type0(...)
- parse_server_type0(...)
- convert_bytes_to_hex(...)

<details>
<summary>Syntax insight</summary>

Keeping these declarations in one header enforces compile-time signature consistency across files. If one side changes and the other does not, the compiler catches it.

</details>

---

## 5) src/utility.c Walkthrough

### send_all
What it does:
- Loops until every byte is sent, because TCP send/write can be partial.

Why needed:
- Prevents truncated frames under load.

### stream_buffer_append + stream_buffer_pop_frame
What they do:
- Maintain per-connection stream state.
- Extract exactly one newline-terminated frame at a time.

Why needed:
- TCP is a byte stream, not message-based.

### build_client_type0 / build_client_type1
What they do:
- Build on-wire client frames with binary type in byte 0.

### build_server_type0
What it does:
- Prepends sender IP and port metadata before payload.

### parse_server_type0
What it does:
- Reconstructs metadata and payload from a received server type-0 frame.

### convert_bytes_to_hex
What it does:
- Converts random bytes into printable uppercase hex payload.

<details>
<summary>Potential pitfall to remember</summary>

If a frame builder or parser disagrees on byte layout, tests fail in confusing ways. Keep builder/parser definitions mirrored.

</details>

---

## 6) src/server.c Walkthrough

## Data model
- client_slot_t: registry entry for active clients.
- msg_node_t: queued message for broadcaster.
- reader_arg_t: per-thread input args.

## Core synchronization
- clients_mutex protects client registry.
- queue_mutex + queue_cond protect queue and wake broadcaster.
- state_mutex protects termination state.

## Main server flow
1. Parse CLI args: ./server <port> <#clients>
2. socket -> bind -> listen
3. Start broadcaster thread
4. accept loop:
- add client to registry
- start detached reader thread

## Reader thread flow
1. recv bytes from one client
2. append to stream buffer
3. pop complete frames
4. if type 0:
- build server type-0 with sender ip/port
- enqueue for broadcaster
5. if type 1:
- mark this client done
- when all expected done, queue final type 1 and stop accept loop

## Broadcaster thread flow
1. dequeue one message from FIFO
2. send to all active clients
3. if message is final type 1, exit broadcaster

<details>
<summary>Why this gives global ordering</summary>

All outbound messages pass through one queue and one sender thread. That serializes delivery order, so clients observe the same sequence.

</details>

<details>
<summary>Why detached reader threads are used</summary>

They avoid join bookkeeping for each client thread and match a thread-per-client pattern adapted from Lab 9.

</details>

---

## 7) src/client.c Walkthrough

## Main flow
1. Parse CLI args:
- ./client <ip> <port> <#messages> <log file>
2. Open log file
3. Connect socket
4. Start receiver thread first
5. Sender loop sends N random type-0 frames
6. Send one type-1 frame
7. Wait for receiver thread and exit

## Receiver thread flow
1. recv bytes continuously
2. stream-frame extraction
3. parse frame type
4. if type 0:
- parse IP/port/payload
- print and log using required alignment format
5. if type 1:
- set shutdown flag and exit thread

## Sender loop details
- getentropy generates random bytes
- convert_bytes_to_hex converts to printable payload
- build_client_type0 wraps payload in protocol frame
- send_all ensures complete frame transmission

<details>
<summary>Why receiver starts before sender</summary>

It minimizes chance of missing early broadcasts while sending starts.

</details>

---

## 8) Lab 9 Reuse: What Was Kept vs Redesigned

Kept from Lab 9 style:
- socket/connect/bind/listen/accept skeleton
- thread-per-client intuition
- mutex-based shared state discipline

Redesigned for A11 correctness:
- binary protocol helpers
- stream framing parser
- global ordered broadcast queue
- two-phase completion logic
- client concurrent send/receive with logging

---

## 9) README Requirement to Code Mapping

1. CLI forms:
- server: parsed in src/server.c main
- client: parsed in src/client.c main

2. AF_INET + bind all interfaces:
- src/server.c socket setup

3. include sender in broadcast + same order:
- src/server.c enqueue_message, broadcaster_thread

4. protocol with type byte + newline:
- src/utility.c builders/parsers + stream helpers

5. sender metadata in server type-0:
- src/utility.c build_server_type0

6. client concurrent send/receive:
- src/client.c receiver thread + sender loop

7. two-phase type-1 completion:
- src/server.c queue_final_type1_if_ready
- src/client.c type-1 send + receiver stop logic

---

## 10) Syntax and Logic Patterns to Remember

1. Safe send loop:
- Loop until total_sent == frame_len.

2. Stream parsing loop:
- append bytes
- pop one frame
- repeat until no complete frame

3. Thread-safe queue:
- producer locks queue, pushes, signals cond
- consumer waits on cond while empty

4. Network format handling:
- carry metadata as binary fields
- convert display values when printing

---

## 11) Quick Self-Check Before Testing

- CMake builds both server and client targets.
- Server accepts expected client count argument.
- Client sends exactly N type-0 then one type-1.
- Each printed client log line follows required alignment format.
- Server adds sender ip/port for type-0 rebroadcast.
- Final type-1 only after all expected clients signaled done.

<details>
<summary>One practical warning</summary>

If any parser/builder mismatch exists (frame boundaries, metadata offsets, or newline placement), tester failures can look like ordering bugs. Verify wire format first.

</details>

---

## 12) Syntax-First Breakdown (What to Notice in C)

This section focuses on the C syntax decisions that are easy quiz targets.

### 12.1 Signed vs unsigned sizes
- ssize_t is used for recv/send/read/write returns.
- size_t is used for buffer lengths and offsets.

Why this matters:
- return values can be negative, lengths cannot.

### 12.2 Pointer and null checks
- Utility helpers guard pointers before using memcpy/memmove.

Why this matters:
- Prevents undefined behavior in shared helpers.

### 12.3 Binary layout operations
- memcpy is used for IP/port field packing and unpacking.

Why this matters:
- Keeps protocol format byte-accurate and avoids text conversion bugs.

### 12.4 Struct and threading syntax
- Thread args are packaged into structs and passed as void*.
- Cast happens inside thread entry function.

Why this matters:
- Standard pthread pattern for multi-argument worker threads.

### 12.5 Synchronization primitives
- pthread_mutex_t and pthread_cond_t coordinate shared queue and state.

Why this matters:
- Correctness under concurrency is a core grading and learning objective.

<details>
<summary>Quiz-style check: identify the right type</summary>

Q: If a function returns bytes sent or -1, should return type be size_t or ssize_t?
A: ssize_t.

</details>

---

## 13) Logic-Level Walkthrough of Key Functions

### utility.c

1. send_all
- Loop invariant: bytes [0..total_sent-1] are already sent.
- Loop exits only when full frame is transmitted or error occurs.

2. stream_buffer_pop_frame
- Searches for newline boundary.
- Copies one full frame out.
- Compacts remaining bytes with memmove.

3. build_server_type0
- Places type at byte 0.
- Writes 4-byte IP then 2-byte port.
- Appends payload and newline.

4. parse_server_type0
- Checks minimum length and delimiters.
- Extracts metadata using fixed offsets.

### server.c

1. enqueue_message / dequeue_message
- Producer-consumer queue for global ordering.

2. reader_thread
- Handles one client stream.
- Parses frames and converts type-0 client frame into server type-0 frame.
- Signals done status on type-1.

3. queue_final_type1_if_ready
- Increments done count once per client.
- Enqueues final type-1 only once.

4. broadcaster_thread
- Only thread that sends queued outbound frames to all clients.

### client.c

1. receiver_thread
- Maintains stream parser state.
- Parses type-0 to printable/loggable fields.
- Stops on type-1.

2. sender loop in main
- getentropy -> hex conversion -> build type-0 -> send_all.
- Sends final type-1 after N messages.

<details>
<summary>Quiz-style check: fixed-offset parsing</summary>

Q: In server type-0 frame, where does payload begin?
A: After 1-byte type + 4-byte IP + 2-byte port.

</details>

---

## 14) Common Failure Patterns and Why

1. Assuming one recv equals one message
- Fails because TCP stream may split/merge payloads.

2. Treating type as ASCII char instead of binary byte
- Breaks protocol interpretation.

3. Sending from multiple threads without ordered serialization
- Breaks "all clients same order" requirement.

4. Not guarding done-count updates
- Can trigger early/duplicate final type-1.

5. Forgetting formatting requirements in client logs
- Functional behavior may pass manually but fail tester checks.

<details>
<summary>Quiz-style check: "what breaks ordering?"</summary>

Q: Which design is more likely to break same-order requirement?
A: Multiple reader threads each broadcasting directly without a shared serialization point.

</details>

---

## 15) Implementation-to-Quiz Mapping

If a quiz asks about protocol:
- Review utility frame builders/parsers.

If a quiz asks about networking lifecycle:
- Review main setup sequence in server/client.

If a quiz asks about concurrency:
- Review mutex ownership and thread roles in server queue/registry.

If a quiz asks about termination:
- Review queue_final_type1_if_ready and client final type-1 send.

If a quiz asks about output formatting:
- Review print_and_log in client.

---

## 16) Short Oral Practice Prompts

1. Explain why newline is needed even with a type byte.
2. Explain why send_all is mandatory for protocol correctness.
3. Explain why queue + single broadcaster helps global order.
4. Explain how two-phase completion prevents premature server shutdown.
5. Explain how client turns binary metadata into human-readable log output.
