# A11 MCQ Quiz Bank (Medium, Lightly Challenging)

Instructions:
- Try answering first without opening the toggle.
- Open the toggle only after committing to an answer.
- Questions are based on README requirements and current implementation files.

Assumption note for size-related questions:
- If needed, assume LP64 model (64-bit Linux/macOS style: int=4, pointer=8, size_t=8).

---

## Q1
Which statement best describes a type-0 message sent from server to client in A11?

A. It contains only payload text and newline.
B. It starts with sender IP string and port string, then type byte.
C. It starts with type byte, then binary IP (4), binary port (2), then payload and newline.
D. It contains no newline terminator.

<details>
<summary>Show answer and explanation</summary>

Correct answer: C

The README requires server type-0 frames to include sender metadata in binary layout after the type byte.

</details>

## Q2
Which command-line form is correct for server startup?

A. ./server <port number> <# of clients>
B. ./server <ip> <port> <log path>
C. ./server <#clients> <#messages>
D. ./server <port> only

<details>
<summary>Show answer and explanation</summary>

Correct answer: A

That exact argument order appears in the README.

</details>

## Q3
Why does send_all exist in utility.c?

A. To avoid needing recv.
B. To convert binary metadata to strings.
C. To guarantee newline is appended.
D. To handle short/partial sends on TCP sockets.

<details>
<summary>Show answer and explanation</summary>

Correct answer: D

A single send may not transmit the whole frame.

</details>

## Q4
What is the most important reason for stream_buffer_pop_frame?

A. It compresses payload bytes.
B. It extracts exactly one newline-terminated frame from a TCP byte stream.
C. It prevents socket creation failures.
D. It randomizes message data.

<details>
<summary>Show answer and explanation</summary>

Correct answer: B

TCP is stream-oriented; framing must be reconstructed.

</details>

## Q5
In two-phase completion, when should server broadcast final type-1?

A. After first client sends type-1.
B. After listener socket opens.
C. After all expected clients have sent type-1.
D. After queue becomes empty once.

<details>
<summary>Show answer and explanation</summary>

Correct answer: C

This is explicitly required by README protocol section.

</details>

## Q6
Which pair is used to print human-readable sender endpoint in client logs?

A. inet_ntop and ntohs
B. htons and inet_pton
C. bind and listen
D. send and recv

<details>
<summary>Show answer and explanation</summary>

Correct answer: A

inet_ntop converts address to text; ntohs converts port to host order.

</details>

## Q7
What is the primary role of the broadcaster thread in server.c?

A. Generate random payloads.
B. Accept new sockets.
C. Parse client CLI arguments.
D. Serialize outbound delivery order by consuming one FIFO queue.

<details>
<summary>Show answer and explanation</summary>

Correct answer: D

One broadcaster over one queue enforces consistent ordering.

</details>

## Q8
Why are reader threads detached in the server accept loop?

A. So they can hold mutexes forever.
B. To avoid requiring pthread_join for each client thread.
C. To disable concurrency.
D. To force synchronous handling.

<details>
<summary>Show answer and explanation</summary>

Correct answer: B

Detached threads clean up themselves on exit.

</details>

## Q9
What does build_client_type1 produce?

A. A frame with text "TYPE1" only.
B. A frame with no type byte.
C. A two-byte frame: type byte 1 then newline.
D. A frame with sender IP + port + newline.

<details>
<summary>Show answer and explanation</summary>

Correct answer: C

Current helper builds [1][\n].

</details>

## Q10
Which is correct TCP server call order?

A. socket -> bind -> listen -> accept
B. bind -> socket -> accept -> listen
C. accept -> socket -> bind -> listen
D. socket -> accept -> listen -> bind

<details>
<summary>Show answer and explanation</summary>

Correct answer: A

Standard server lifecycle.

</details>

## Q11
In client.c, why start receiver before sender loop?

A. To reduce compile warnings.
B. To avoid missing incoming broadcasts while sending starts.
C. To convert ports with htons.
D. To detach broadcaster.

<details>
<summary>Show answer and explanation</summary>

Correct answer: B

Concurrent receive is required; starting early reduces race windows.

</details>

## Q12
What does parse_server_type0 validate first?

A. That payload contains only uppercase hex.
B. That socket is in listen mode.
C. That frame size is exactly 1024.
D. That frame is large enough and starts with type 0 and ends with newline.

<details>
<summary>Show answer and explanation</summary>

Correct answer: D

It checks structural validity before extracting fields.

</details>

## Q13
Why is queue_mutex paired with queue_cond?

A. To parse CLI strings faster.
B. To convert sockaddr_in to string.
C. To let broadcaster sleep when queue is empty and wake when producers enqueue.
D. To enforce client log file format.

<details>
<summary>Show answer and explanation</summary>

Correct answer: C

Condition variable handles producer/consumer synchronization.

</details>

## Q14
Which command-line form is correct for client startup?

A. ./client <IP address> <port number> <# of messages> <log file path>
B. ./client <port> <#clients>
C. ./client <ip> <port> only
D. ./client <tester path> <test #>

<details>
<summary>Show answer and explanation</summary>

Correct answer: A

Matches README exactly.

</details>

## Q15
What problem appears if code assumes one recv call equals one full message?

A. Faster execution always.
B. No difference on TCP.
C. Better alignment for structs.
D. Framing bugs, because TCP can split or merge application messages.

<details>
<summary>Show answer and explanation</summary>

Correct answer: D

TCP delivers bytes, not message boundaries.

</details>

## Q16
Which statement about type bytes is correct in this implementation?

A. Type is stored as ASCII character '0' or '1' on wire.
B. Type is stored as binary byte value 0 or 1 on wire.
C. Type is omitted for server messages.
D. Type appears after payload.

<details>
<summary>Show answer and explanation</summary>

Correct answer: B

Builders place numeric byte 0 or 1 in first byte.

</details>

## Q17
Assuming LP64, what is sizeof(receiver_ctx_t) most likely?
(receiver_ctx_t has int sockfd; FILE *log_fp; sig_atomic_t *shutdown_requested;)

A. 24 bytes
B. 12 bytes
C. 16 bytes
D. 32 bytes

<details>
<summary>Show answer and explanation</summary>

Correct answer: A

Likely layout: int(4) + padding(4) + pointer(8) + pointer(8) = 24.

</details>

## Q18
Assuming LP64, what is sizeof(msg_node_t) most likely?
(msg_node_t has uint8_t data[1024]; size_t len; pointer next;)

A. 1032 bytes
B. 1048 bytes
C. 1040 bytes
D. 1024 bytes

<details>
<summary>Show answer and explanation</summary>

Correct answer: C

1024 + 8 + 8 = 1040, already aligned to 8.

</details>

## Q19
What is the main reason server snapshots active FDs before broadcast sends?

A. To avoid inet_pton failures.
B. To avoid holding client registry lock while performing potentially blocking send operations.
C. To increase random entropy.
D. To enforce test ordering by test number.

<details>
<summary>Show answer and explanation</summary>

Correct answer: B

Holding locks during blocking I/O can create contention/deadlock risk.

</details>

## Q20
Which README grading rule is directly linked to CMake target names?

A. Missing README comments gives zero.
B. Not using pthread gives zero.
C. Wrong log level gives automatic zero.
D. Not generating required executables named client and server can result in zero.

<details>
<summary>Show answer and explanation</summary>

Correct answer: D

Executable names and generation are explicitly graded constraints.

</details>

## Q21
Which helper is specifically for converting fuzz bytes to printable payload text?

A. convert_bytes_to_hex
B. parse_server_type0
C. stream_buffer_pop_frame
D. build_client_type1

<details>
<summary>Show answer and explanation</summary>

Correct answer: A

It transforms raw random bytes into uppercase hex string.

</details>

## Q22
In the current server design, what triggers accept loop termination?

A. Queue has more than 100 messages.
B. Broadcaster thread exits first.
C. Final type-1 gets queued, accepting flag is cleared, and listening socket is closed.
D. First client disconnects.

<details>
<summary>Show answer and explanation</summary>

Correct answer: C

queue_final_type1_if_ready flips acceptance state and closes listen socket.

</details>

## Q23
Which command is for testing your client executable against a tester-run server harness?

A. ./server-tester <IP> <port> ...
B. ./client <ip> <port> ...
C. minimum_checker
D. ./client-tester <client executable path> <client log prefix> <port> <#clients> <#messages> <test #>

<details>
<summary>Show answer and explanation</summary>

Correct answer: D

client-tester launches and validates your client behavior.

</details>

## Q24
Why is newline still required even though messages begin with a type byte?

A. Type byte fully determines payload length.
B. Newline provides frame boundary for variable-length payloads in stream parsing.
C. Newline is only for printing convenience.
D. Newline is ignored by parser.

<details>
<summary>Show answer and explanation</summary>

Correct answer: B

Without a delimiter or explicit length field, variable payload framing would be ambiguous.

</details>

---

## Bonus self-drill

Before opening any answers, try explaining out loud:
1. Why global ordering is a server architecture concern, not a client print concern.
2. Why send_all and stream buffer helpers are both required.
3. Why type-1 is a protocol state transition, not just another chat payload.
