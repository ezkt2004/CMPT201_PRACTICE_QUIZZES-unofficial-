# 📚 CMPT 201 — Comprehensive Midterm Study Notes
### Spring 2026 | Lectures 1–6 + Brief Lecture 7

> Built from: lecture slides, activities, labs 1–5, sample midterm format, and professor hints.
> **Goal:** Everything testable in one place. Code snippets, algorithms, key one-liners, exam traps.

---

## 📋 Table of Contents

1. [L1 — Tour of Computer Systems](#-l1--tour-of-computer-systems)
2. [L2.1 — Processes & sleep()](#-l21--processes--sleep--man-pages)
3. [L2.2 — fork(), exec(), wait(), errno](#-l22--fork-exec-wait-errno)
4. [L3 — Signals & Function Pointers](#-l3--signals--function-pointers)
5. [L4 — Scheduling Algorithms](#-l4--scheduling-algorithms)
6. [L5 — Memory Management](#-l5--memory-management)
7. [L6 — Virtual Memory](#-l6--virtual-memory)
8. [L7 — Threads (Brief)](#-l7--threads-brief)
9. [Labs Quick Reference](#-labs-quick-reference)
10. [Critical Code Templates](#-critical-code-templates)
11. [Quick-Fire Exam One-Liners](#-quick-fire-exam-one-liners)

---

## 🖥️ L1 — Tour of Computer Systems

### OS Stack (Three Layers)

```
┌─────────────────────────────┐
│        Applications         │  ← User programs (Python, shell, browsers)
├─────────────────────────────┤
│          Kernel             │  ← Manages hardware, processes, memory
├─────────────────────────────┤
│          Hardware           │  ← CPU, RAM, I/O devices
└─────────────────────────────┘
```

### Von Neumann Architecture

- **Fundamental design model** for modern computers
- CPU **fetches data from memory** → provides it to CPU for computation
- Two fundamental components:
  - **Computation** → CPU
  - **Data** → Memory (RAM + storage)

### Memory Hierarchy (fastest → slowest, smallest → largest)

```
Registers  →  Cache (L1/L2/L3)  →  RAM  →  SSD  →  HDD  →  Tape
(in-CPU)      (on-chip)           (DRAM)   (flash) (magnetic)(archival)
```

> **Exam one-liner:** Want the best of all worlds — fast access, large capacity, low price. Memory hierarchy intelligently moves data from large-slow devices into small-fast ones.

### CPU Architecture Key Facts

| Topic | Detail |
|-------|--------|
| Moore's Law phase 1 | ~1970–2005: transistor count doubled → faster single cores |
| Moore's Law phase 2 | ~2005–present: shift to **multiple cores** (heat/power limits hit) |
| ISA | Instruction Set Architecture — defines the set of instructions a CPU understands |
| x86 vs ARM | Two different ISAs — incompatible machine code |
| 32-bit CPU | Registers/pointers = 4 bytes. Max addressable memory = 2^32 = ~4 GB |
| 64-bit CPU | Registers/pointers = 8 bytes. Vast address space (2^64) |

### Kernel Mode vs User Mode

| | Kernel Mode (Ring 0) | User Mode |
|--|--|--|
| Access | Full hardware & memory | Restricted — no privileged instructions |
| Who uses it | Kernel code | All user applications (including root) |
| Direct HW access | ✅ Yes | ❌ No |
| Access kernel memory | ✅ Yes | ❌ No |

> **Exam trap:** A **superuser (root)** is a *user* with administrative privileges — they still run in **user mode**. Kernel mode ≠ superuser.

### Why User Mode Exists

- **Protection/isolation** — prevents buggy/malicious user programs from corrupting kernel or other processes
- **Abstraction** — provides a clean interface (syscalls) rather than raw hardware access
- **Stability** — a crash in user mode doesn't crash the kernel

### Kernel Roles

1. **Resource management** — mediates hardware access among competing processes
2. **Process control** — starting, stopping, scheduling processes
3. **Protection** — isolating processes from each other and from kernel

### Event-Driven Kernel

The kernel is **reactive** — it responds to events rather than running continuously.

Three types of events:
1. **Hardware interrupt** — mouse click, keyboard press, timer
2. **Syscall** — `fork()`, `write()`, `read()` from a user program
3. **Signal** — `SIGINT` (Ctrl+C), `SIGTERM`, `SIGSEGV` (segfault)

### Systems Programming

- **Low-level programming** that directly interacts with hardware or OS via syscall interface
- Requires languages like C, C++, Rust — Python/Java **cannot** do raw memory access
- Examples: OS kernels, device drivers, memory allocators, shells

---

## ⚙️ L2.1 — Processes & sleep(), Man Pages

### Program vs Process

| | Program | Process |
|--|--|--|
| What is it | Compiled executable file on disk | A **running** program |
| State | Static | Dynamic (has memory, CPU state) |
| Life | Until deleted | From `fork()`/run until exit |

### Program in Memory — What the OS Does at Launch

1. Creates a **memory space** for the process
2. **Loads machine code** from the executable on disk into memory
3. Sets up memory for data (stack, heap, globals)
4. Starts **executing** from the program's entry point

### Reading Man Pages — Procedure

1. **Synopsis** — function signature, header files, return type
2. **Description** — what does it do? Skim for relevant part
3. **Return value** — what does it return on success vs failure?
4. **Errors** — what are possible `errno` values?
5. **Feature Test Macro** — does it need `#define _POSIX_C_SOURCE`?

| Section | Covers | Example |
|---------|--------|---------|
| `man 1` | Shell commands | `man 1 ls` |
| `man 2` | System calls | `man 2 fork` |
| `man 3` | C stdlib functions | `man 3 printf` |

### `sleep()` Activity Key Concepts

```c
#include <unistd.h>
unsigned int sleep(unsigned int seconds);
// Returns: 0 if completed, or remaining seconds if interrupted by signal
```

> **Why `fflush(stdout)` with `printf`?** `printf` is buffered — output may not appear immediately. `fflush` forces the buffer to flush. `write()` is unbuffered and appears immediately.

### C Pointers Review

```c
// Pointer to int
int x = 5;
int *p = &x;   // p holds address of x
*p = 10;       // dereference — changes x to 10

// Pointer to pointer (output parameter pattern):
void set_ptr(char **ppstr) {
    *ppstr = "hello";   // sets the caller's pointer to point to "hello"
}
char *str = NULL;
set_ptr(&str);  // pass address of str
// str now points to "hello"
```

> **Key:** `char **ppstr` — double pointer. Caller passes `&ptr` (address of their pointer). Function uses `*ppstr = ...` to set where the caller's pointer points. Used in `strtok_r` and `getline`.

---

## 🔧 L2.2 — fork(), exec(), wait(), errno

### `fork()`

```c
#include <unistd.h>
pid_t fork(void);
```

| Return Value | Meaning |
|--|--|
| `< 0` | Error — child not created; check `errno` |
| `== 0` | You are the **child** |
| `> 0` | You are the **parent**; value = child's PID |

**What fork does:** Creates an **identical copy** of the calling process. Both parent and child continue executing from the line after `fork()`.

```c
pid_t pid = fork();
if (pid < 0) {
    perror("fork");
    exit(1);
} else if (pid == 0) {
    // CHILD process code here
    printf("Child: my PID = %d, parent PID = %d\n", getpid(), getppid());
} else {
    // PARENT process code here
    printf("Parent: child PID = %d\n", pid);
}
```

> **Key:** Up to `fork()`, one process. After `fork()`, two **separate copies** running concurrently.

### `exec()` Family

```c
#include <unistd.h>
int execvp(const char *file, char *const argv[]);
// file:  program name (searched in PATH)
// argv:  argument array, MUST end with NULL
// Returns: NEVER on success (process replaced), -1 on failure
```

**What exec does:** Replaces the current process's **entire memory image** with a new program. Code after `execvp()` only runs if exec **fails**.

```c
char *args[] = {"ls", "-l", "-a", NULL};  // NULL-terminated!
execvp("ls", args);
// If we reach here, execvp FAILED:
perror("execvp");
exit(EXIT_FAILURE);
```

> **Exam trap:** After successful `exec()`, the calling process's code no longer exists in memory. The function never returns.

### `waitpid()`

```c
#include <sys/wait.h>
pid_t waitpid(pid_t pid, int *wstatus, int options);
```

| `pid` argument | Meaning |
|--|--|
| `> 0` (specific PID) | Wait for that **specific child** |
| `-1` | Wait for **any child** |
| `0` | Any child in same process group |

| `options` | Meaning |
|--|--|
| `0` | **Block** until specified child exits |
| `WNOHANG` | Return **immediately** if no child has exited (non-blocking) |

**Status macros after `waitpid()`:**

```c
int wstatus;
waitpid(pid, &wstatus, 0);

WIFEXITED(wstatus)    // true if child exited normally
WEXITSTATUS(wstatus)  // exit code (only valid if WIFEXITED)
WIFSIGNALED(wstatus)  // true if child killed by signal
WTERMSIG(wstatus)     // signal number that killed child
```

### Zombie vs Orphan

| | Caused by | State |
|--|--|--|
| **Zombie** | Child exits before parent calls `wait()` | Child finished, but PID entry stays in process table |
| **Orphan** | Parent exits before child finishes | Child re-parented to `init` (PID 1), which will reap it |

> **Zombie cleanup:** The OS keeps zombie entries so the parent can retrieve the exit status. Once `waitpid()` is called, entry is removed.

### `errno`

```c
#include <errno.h>
// errno is set automatically by syscalls/stdlib on error
// Common values:
// EINTR  — interrupted by signal (not a real error - retry the call)
// ECHILD — no children to wait on (normal loop exit condition)
// EAGAIN — try again (resource temporarily unavailable)
// ENOMEM — out of memory
// ENOENT — no such file or directory
```

```c
// Pattern: check errno after syscall failure
if (syscall_result == -1) {
    if (errno == EINTR) {
        continue;  // just a signal, not a real error
    }
    perror("syscall name");  // prints human-readable error
    exit(1);
}
```

### `getline()` — Lab 1 Key Function

```c
#include <stdio.h>
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
// lineptr: pointer to buffer pointer (can be NULL — getline will malloc)
// n: pointer to buffer size (set to 0 if lineptr is NULL)
// stream: file to read from (stdin for keyboard)
// Returns: characters read (including '\n'), or -1 on error/EOF
```

```c
char *buf = NULL;
size_t buf_size = 0;
ssize_t n = getline(&buf, &buf_size, stdin);
if (n == -1) { perror("getline"); }
// ... use buf ...
free(buf);  // ALWAYS free — getline malloc'd this!
```

> **Note:** `getline()` **grows its buffer** dynamically — unlike `fgets()` which truncates. Must `free()` the buffer.

### `strtok_r()` — Lab 1 Key Function

```c
char *strtok_r(char *str, const char *delim, char **saveptr);
// First call: str = your input string
// Subsequent calls: str = NULL (saveptr tracks position)
// MODIFIES the input string (replaces delimiters with '\0')
```

```c
char *saveptr;
char *token = strtok_r(buf, " \n", &saveptr);   // first call
while (token != NULL) {
    printf("%s\n", token);
    token = strtok_r(NULL, " \n", &saveptr);     // subsequent: NULL!
}
```

> **Exam trap:** `strtok()` uses hidden static state → not re-entrant. `strtok_r()` uses explicit `saveptr` → re-entrant and thread-safe.
> **Trap 2:** `strtok_r()` **destroys** the original string. Save a copy before tokenizing if you need the original.

---

## 📡 L3 — Signals & Function Pointers

### Function Pointers

```c
// Syntax: return_type (*pointer_name)(param_types) = function_name;
int (*fp)(char a, int b) = myFunc;  // fp points to myFunc

// Call via pointer:
int result = fp('x', 42);

// In a struct (like sigaction):
struct sigaction {
    void (*sa_handler)(int);   // function pointer to signal handler
    sigset_t sa_mask;
    int sa_flags;
    // ...
};
```

> **Why function pointers?** They allow passing a function's address as a value — e.g., giving the kernel the address of your signal handler so it can call it when a signal arrives.

### Signals Overview

- Signals are **notifications** sent to processes by the OS, kernel, or other processes
- Common signals:

| Signal | Number | Cause | Default Action |
|--------|--------|-------|----------------|
| `SIGINT` | 2 | Ctrl+C | Terminate |
| `SIGTERM` | 15 | `kill` command | Terminate |
| `SIGKILL` | 9 | Force kill (cannot be caught!) | Terminate |
| `SIGSEGV` | 11 | Segmentation fault | Terminate + core dump |
| `SIGCHLD` | 17 | Child state changed | Ignore |

### `sigaction()` — Complete Pattern

```c
#define _POSIX_C_SOURCE 200809L
#include <signal.h>
#include <unistd.h>
#include <string.h>

// The signal handler — MUST be async-signal-safe
void my_handler(int signum) {
    const char *msg = "Signal caught!\n";
    write(STDOUT_FILENO, msg, strlen(msg));  // write() NOT printf()!
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = my_handler;   // assign handler function pointer
    sigemptyset(&sa.sa_mask);     // don't block other signals during handler
    sa.sa_flags = 0;              // no special flags needed
    sigaction(SIGINT, &sa, NULL); // install for SIGINT (Ctrl+C)

    while (1) { sleep(1); }
    return 0;
}
```

### Async-Signal-Safety

> **Golden Rule:** Inside a signal handler, only use **async-signal-safe** functions. Check `man signal-safety`.

| ✅ Signal-SAFE | ❌ NOT Signal-Safe |
|---|---|
| `write()` | `printf()` |
| `read()` | `fprintf()`, `puts()` |
| `_exit()` | `malloc()`, `free()` |
| `getpid()` | `snprintf()` |
| `strlen()` | `strerror()` |
| `kill()` | Most stdio functions |

**Why?** Non-safe functions use internal locks/buffers. If a signal interrupts a function mid-lock and the handler calls the same function → **deadlock or corruption**.

### SIGINT and EINTR Pattern

```c
input_read = read(STDIN_FILENO, input, sizeof(input) - 1);
if (input_read == -1) {
    if (errno == EINTR) {
        continue;   // Signal interrupted read — not a real error, retry
    }
    // Real error:
    write(STDERR_FILENO, "shell: unable to read command\n", 30);
    continue;
}
```

### `kill()` and `raise()`

```c
kill(pid, SIGINT);   // send SIGINT to process with given pid
raise(SIGINT);       // send signal to yourself (equivalent to kill(getpid(), sig))
```

---

## 📊 L4 — Scheduling Algorithms

### Key Terminology

| Term | Definition |
|------|-----------|
| **CPU scheduling** | Deciding which process runs next on a core |
| **Context switch** | Stop one process, start/resume another (has overhead!) |
| **Ready queue** | Processes waiting to run (loaded in memory, not blocking on I/O) |
| **I/O queue** | Processes waiting for I/O to complete |
| **Quantum** | Time slice each process gets in Round Robin |
| **Preemptive** | Scheduler can interrupt a running process |
| **Non-preemptive** | Process runs until it voluntarily yields or finishes |

### Scheduling Criteria

- **Maximize:** CPU utilization (keep CPU busy)
- **Minimize:**
  - **Turnaround time** = completion time − arrival time
  - **Waiting time** = turnaround time − burst time
  - **Response time** = time from submission to first response

### FCFS — First Come, First Served

- **Non-preemptive** — runs in order of arrival
- **Problem:** Convoy effect — a long process blocks all shorter ones behind it
- **Gantt example:**

```
Arrival: P1=0(burst 24), P2=1(burst 3), P3=2(burst 3)

[P1: 0-24][P2: 24-27][P3: 27-30]

Waiting: P1=0, P2=23, P3=25 → Average = 16 ms  ← terrible!
```

### SJF — Shortest Job First

- **Non-preemptive** — pick shortest burst time from ready queue
- Optimal for **average waiting time** among non-preemptive algorithms
- Problem: requires knowing burst time in advance

### SRTF — Shortest Remaining Time First

- **Preemptive** version of SJF
- At every arrival, compare new process's burst with current remaining time — preempt if shorter
- **Always results in better or equal average waiting time than SJF**

**SRTF calculation method:**
1. Start running the process with shortest burst
2. When new process arrives: compare arriving burst vs current remaining
3. If arriving < remaining → preempt; else continue
4. Waiting time = (finish time) − (arrival time) − (burst time)

### Round Robin (RR)

- **Preemptive** — each process gets a fixed quantum; preempted when quantum expires
- Process added to back of ready queue when preempted or when it arrives
- **If quantum → ∞**: effectively FCFS
- **If quantum → 0**: pure context switch overhead, no work done

**RR Gantt trace approach:**
1. Maintain a ready queue
2. At each time point: add arriving processes to ready queue
3. Run current process for min(quantum, remaining_burst)
4. If not finished: add back to queue

### Priority Scheduling

- Pick highest-priority process from ready queue
- Can be preemptive or non-preemptive
- **Problem: starvation** — low-priority processes may never run
- **Aging** solution: gradually increase priority of waiting processes

### Multilevel Queue Scheduling

- Multiple queues, one per process category (system, foreground, background)
- Each queue has a priority and uses its own internal algorithm
- **Weighted Round Robin** across queues: higher-priority queues get more CPU time
- Avoids the starvation problem of pure priority scheduling

### Summary Table

| Algorithm | Preemptive | Best For | Problem |
|-----------|-----------|----------|---------|
| FCFS | No | Simple batch | Convoy effect |
| SJF | No | Minimize avg wait | Needs future knowledge |
| SRTF | Yes | Optimal avg wait | Needs future knowledge |
| RR | Yes | Interactive/fair | Context switch overhead |
| Priority | Either | Real-time deadlines | Starvation |
| Multilevel Queue | Either | Mixed workloads | Complex |

---

## 💾 L5 — Memory Management

### Heap Allocation Overview

```
User program → calls malloc() → memory allocator (user space)
                                     ↓ (when heap exhausted)
                               calls sbrk() → OS kernel → expands heap
```

### `sbrk()` — The Heap Syscall

```c
#include <unistd.h>
void *sbrk(intptr_t increment);
// Moves the program break up by `increment` bytes
// Returns: PREVIOUS break (= start of newly allocated region)
// Returns (void*)-1 on failure
```

**Program break:** The first address after the end of the BSS (uninitialized data) segment — marks where the heap ends.

```c
// Get 256 bytes of raw heap memory:
void *heap_start = sbrk(256);
if (heap_start == (void *)-1) { /* error */ }
// Now heap_start points to 256 bytes of usable memory
```

> **Don't call sbrk() for every allocation** — syscall overhead! Get a big chunk once, manage it internally.

> **Don't mix sbrk() and malloc()** — they both manage the program break; mixing causes conflicts. Use `write()` not `printf()` when using `sbrk()` directly (Lab 4).

### Memory Layout — Virtual Address Space

```
High addresses
┌───────────────────┐
│      Kernel       │  ← inaccessible from user mode
├───────────────────┤
│  Stack (↓ grows)  │  ← local variables, function call frames
│                   │
│   [gap / mmap]    │  ← memory-mapped files, shared libraries
│                   │
│   Heap (↑ grows)  │  ← dynamic allocation (malloc/sbrk)
├───────────────────┤
│  BSS (uninit)     │  ← uninitialized global/static variables (zero-filled)
├───────────────────┤
│  Data (init)      │  ← initialized global/static variables
├───────────────────┤
│  Text (code)      │  ← program instructions (read-only)
└───────────────────┘
Low addresses (0)
```

### Memory Allocator Concepts

**Allocator task:** Manage the heap. When `malloc(n)` is called, find a free region of ≥ n bytes and return a pointer to it.

**Free list:** A **linked list of free memory blocks**. Each free block has a header:

```c
struct header {
    uint64_t size;       // size of this block (including header)
    struct header *next; // pointer to next free block
};
```

**In-place linked list:** The header is stored **inside** the free block itself — no separate allocation needed.

### Allocation Strategies (Finding a Free Block)

| Strategy | Algorithm | Best For |
|----------|-----------|---------|
| **First-fit** | Return first block with size ≥ requested | Fast |
| **Best-fit** | Return smallest block with size ≥ requested | Less wasted space |
| **Worst-fit** | Return largest block | Leaves large remainders (useful for future?) |

**Lab 5 code pattern:**

```c
// First Fit:
struct header *curr = free_list_ptr;
while (curr != NULL) {
    if (curr->size >= requested_size) return curr->id;
    curr = curr->next;
}
return -1;

// Best Fit:
struct header *curr = free_list_ptr;
int best_id = -1;
uint64_t best_size = UINT64_MAX;
while (curr != NULL) {
    if (curr->size >= requested_size && curr->size < best_size) {
        best_size = curr->size;
        best_id = curr->id;
    }
    curr = curr->next;
}
return best_id;

// Worst Fit:
struct header *curr = free_list_ptr;
int worst_id = -1;
uint64_t worst_size = 0;
while (curr != NULL) {
    if (curr->size >= requested_size && curr->size > worst_size) {
        worst_size = curr->size;
        worst_id = curr->id;
    }
    curr = curr->next;
}
return worst_id;
```

### Fragmentation

| Type | Description | Caused by |
|------|-------------|-----------|
| **External fragmentation** | Total free memory is enough but no single contiguous block is big enough | Many small scattered free blocks |
| **Internal fragmentation** | Allocated block is larger than needed | Alignment requirements, block size rounding |

### Coalescing

**Problem:** After freeing blocks, adjacent free blocks accumulate. Individually too small for large requests.

**Solution:** When freeing a block, check if adjacent blocks are also free → **merge them** into a larger block.

```
Before: [FREE:16][USED:8][FREE:12] → can't serve malloc(24)
After freeing middle:  [FREE:16][FREE:8][FREE:12]
After coalescing: [FREE:36]  → can now serve malloc(24)!
```

---

## 🗺️ L6 — Virtual Memory

### Why Virtual Memory?

**Problem with early systems:** Processes used raw physical addresses → no isolation, limited memory per program.

**Virtual memory solution:**
1. **Physical memory sharing** — multiple processes share physical RAM transparently
2. **Isolation** — each process only sees its own virtual address space

### Virtual Address Space

- Each process gets its own **virtual address space** ranging from 0 to address-max (2^64-1 for 64-bit)
- Each address in this space is a **virtual address** (not a physical address)
- User processes **only** deal with virtual addresses, never physical

> **Key insight:** Two processes can have virtual address `0x1000` but they map to different physical addresses — complete isolation!

### Address Translation

**Mechanism:** Maps virtual address → physical address at runtime.

```
CPU generates virtual address
        ↓
   MMU (hardware) + Page Table (kernel data structure)
        ↓
   Physical address in RAM
```

### Paging

**Concept:** Virtual address space divided into fixed-size **pages**; physical memory divided into equal-size **page frames**.

| Term | Definition |
|------|-----------|
| **Page** | Fixed-size region of virtual address space |
| **Page frame** (frame) | Fixed-size region of physical memory |
| **Page table** | Kernel data structure mapping virtual page numbers → physical frame numbers |
| **MMU** | Memory Management Unit — hardware that does the translation |

**Address format in paging:**

```
Virtual address = [ VPN (Virtual Page Number) | Offset ]

Example: page size = 4KB = 2^12
→ 12 offset bits
→ remaining bits = VPN bits

For 32-bit address space: 32 - 12 = 20 VPN bits → 2^20 = 1M page table entries
```

**Page table size calculation:**

```
number_of_pages = virtual_address_space_size / page_size

Example:
- Virtual space: 256 KB = 2^18 bytes → 18-bit addresses
- Page size: 4 KB = 2^12 → 12 offset bits
- VPN bits: 18 - 12 = 6 bits
- Number of page table entries: 2^6 = 64
```

**Address translation example:**

```
Page size = 4 KB = 4096 bytes

Virtual address = 0x5A3F = 23103 decimal
VPN    = 23103 / 4096 = 5 (page 5)
Offset = 23103 % 4096 = 3007 bytes into page 5

→ Look up page 5 in page table → get physical frame number
→ Physical address = frame_base + 3007
```

### Segmentation

- Divides virtual address space into **variable-size, semantically meaningful segments** (text, data, stack, heap)
- Each segment has a base address and size (limit)
- **Virtual address = [segment number | offset within segment]**
- **Suffers from external fragmentation** (variable sizes leave holes)
- Modern OSes use **paging** (not segmentation) — paging avoids external fragmentation

### Demand Paging & Swapping

| Concept | Description |
|---------|-------------|
| **Demand paging** | Pages loaded into physical memory **only when accessed** (not all at once) |
| **Page fault** | Process accesses a virtual page not currently in physical memory → trap to kernel |
| **Swapping** | When physical memory is full, evict (swap out) a page to disk to make room |

**Why does demand paging work?** → **Temporal and spatial locality** — programs tend to access the same memory regions repeatedly, so most pages are never needed at a given time.

**Page fault handling sequence:**
1. MMU detects page not in memory → hardware trap (page fault) to kernel
2. Kernel finds the page on disk
3. Kernel evicts a page from physical memory if needed (page replacement algorithm)
4. Kernel loads the required page into the free frame
5. Updates page table
6. **Resumes the faulting instruction** — process doesn't know this happened!

### Page Table Size Problem

For 64-bit addresses with 4 KB pages: 2^64 / 4096 = 2^52 entries — too large!

**Solution: Multi-level Page Tables** — hierarchical page tables that avoid allocating entries for unused regions.

---

## 🧵 L7 — Threads (Brief)

### Thread vs Process

| | Process | Thread |
|--|--|--|
| Address space | Separate for each process | **Shared** (text, data, BSS, heap shared) |
| Stack | Own stack | Each thread has **own stack** |
| Creation overhead | High (copy full address space) | Low (no new address space) |
| Data sharing | Requires IPC | Easy — share globals/heap directly |

```
Process memory layout with 2 threads:
┌──────────────┐
│ Main thread  │ ← own stack
│    stack     │
├──────────────┤
│ Thread 1     │ ← own stack
│    stack     │
├──────────────┤
│  Heap        │ ← SHARED
├──────────────┤
│  BSS         │ ← SHARED
├──────────────┤
│  Data        │ ← SHARED
├──────────────┤
│  Text        │ ← SHARED
└──────────────┘
```

### `pthread_create()` Pattern

```c
#include <pthread.h>
// Compile with: clang -pthread program.c

// Thread function signature:
void *thread_func(void *arg) {
    char *str = (char *) arg;    // cast the void* arg
    printf("Thread: %s\n", str);
    return (void *) (long) strlen(str);  // return value as void*
}

int main() {
    pthread_t tid;
    void *retval;

    // Create thread: (thread_id, attr, function_ptr, argument)
    pthread_create(&tid, NULL, thread_func, (void *)"hello");

    // Wait for thread, get return value:
    pthread_join(tid, &retval);
    printf("Thread returned: %ld\n", (long) retval);
    return 0;
}
```

### Data Race Problem

```c
static int cnt = 0;  // shared global

void *inc(void *arg) {
    for (int i = 0; i < 10000000; i++) {
        cnt++;   // NOT atomic: load → add → store (3 steps!)
    }
    return NULL;
}
```

**Problem:** `cnt++` is NOT atomic. Two threads can both load `cnt=5`, both add to get `6`, and both store `6` — **one increment is lost**. Output is non-deterministic and less than expected.

**Data race:** Different threads race to update the same data and overwrite each other's result.

> **Key definition:** A **race condition** is when correctness depends on the timing/order of operations. A **data race** is a specific type where threads access shared data without synchronization.

---

## 🔬 Labs Quick Reference

### Lab 1 — `getline()` + `strtok_r()`

```c
// Full pattern:
char *buf = NULL;
size_t size = 0;
ssize_t n = getline(&buf, &size, stdin);
// n includes '\n'; n == -1 on EOF/error
// strtok_r needs NULL on 2nd+ call
char *saveptr;
for (char *tok = strtok_r(buf, " \n", &saveptr);
     tok != NULL;
     tok = strtok_r(NULL, " \n", &saveptr)) {
    printf("%s\n", tok);
}
free(buf);  // mandatory!
```

### Lab 2 — `fork() + execl() + waitpid()`

```c
pid_t pid = fork();
if (pid == 0) {
    // child: run a program
    execl("/usr/bin/ls", "ls", "-a", NULL);  // first arg = path, second = argv[0]
    perror("exec"); exit(1);
} else {
    int status;
    waitpid(pid, &status, 0);
}
```

> **`execl()` vs `execvp()`:** `execl` takes variadic args ending with NULL; `execvp` takes an array and searches PATH.

### Lab 3 — History Ring Buffer

```c
#define HISTORY_SIZE 5
char *history[HISTORY_SIZE];  // array of string pointers
int history_count = 0;

void add(char *line) {
    int idx = history_count % HISTORY_SIZE;
    free(history[idx]);          // free old entry if overwriting
    history[idx] = strdup(line); // duplicate the string (malloc internally)
    history_count++;
}
```

> **Memory management:** `getline()` allocates strings. When overwriting a history slot, `free()` the old pointer first before overwriting.

### Lab 4 — `sbrk()` + struct in raw memory

```c
struct header {
    uint64_t size;
    struct header *next;
};

// Allocate 256 bytes
struct header *first = (struct header *) sbrk(256);
// Second block at offset 128
struct header *second = (struct header *)((char *)first + 128);

first->size = 128;
first->next = NULL;
second->size = 128;
second->next = first;

// Fill data (skip header with pointer arithmetic):
char *first_data = (char *)(first + 1);   // skip header
memset(first_data, 0, 128 - sizeof(struct header));
```

### Lab 5 — Allocation Algorithm Comparison

Free list: sizes [6, 12, 24, 8, 4], request = 7

| Algorithm | Selected | Why |
|-----------|---------|-----|
| **First-fit** | Block 2 (size 12) | First block ≥ 7 |
| **Best-fit** | Block 4 (size 8) | Smallest block ≥ 7 |
| **Worst-fit** | Block 3 (size 24) | Largest available |

---

## 💡 Critical Code Templates

### Template 1: Fork + Exec + Wait (foreground)

```c
pid_t pid = fork();
if (pid < 0) { perror("fork"); exit(1); }
else if (pid == 0) {
    char *args[] = {"ls", "-l", NULL};
    execvp(args[0], args);
    perror("execvp"); exit(EXIT_FAILURE);
} else {
    int wstatus;
    if (waitpid(pid, &wstatus, 0) == -1) perror("waitpid");
    if (WIFEXITED(wstatus)) printf("Exit: %d\n", WEXITSTATUS(wstatus));
}
```

### Template 2: Zombie Cleanup Loop

```c
void cleanup_zombies() {
    int wstatus;
    pid_t pid;
    while ((pid = waitpid(-1, &wstatus, WNOHANG)) > 0) { /* reap */ }
    if (pid == -1 && errno != ECHILD) perror("waitpid");
}
```

### Template 3: Signal Handler (Complete)

```c
static char msg[] = "Signal caught!\n";
void handler(int sig) {
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);
}
// In main():
struct sigaction sa = {0};
sa.sa_handler = handler;
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;
sigaction(SIGINT, &sa, NULL);
```

### Template 4: SRTF Calculation

```
Input: Process, burst, arrival
Step 1: At each arrival time, compare new burst vs current remaining
Step 2: Preempt if new < current remaining
Step 3: Waiting = (finish) - (arrival) - (burst)
Step 4: Average = sum(waiting) / number_of_processes
```

### Template 5: Paging Math

```
Given: virtual space = X bytes, page size = P bytes
offset bits = log2(P)
VPN bits = log2(X) - offset bits  (or: log2(X/P))
page table entries = X / P = 2^(VPN bits)

For address VA:
VPN    = VA / P         (integer division)
Offset = VA % P         (modulo)
```

---

## ⚡ Quick-Fire Exam One-Liners

### Computer Systems
- **Von Neumann:** CPU fetches from memory for computation; two components: CPU (computation) + memory (data)
- **Moore's Law shift:** ~2005 moved from faster single cores to multiple cores due to heat/power limits
- **Kernel mode = Ring 0** = full hardware privilege; user mode = restricted, no privileged instructions
- **Superuser (root) ≠ kernel mode** — root still runs in user mode
- **Syscall interface:** the boundary where user programs ask the kernel to do privileged operations
- **Event-driven kernel:** responds to hardware interrupts, syscalls, and signals

### Processes
- **Program** = file on disk; **Process** = running program with memory space
- **fork()** returns `< 0` (error), `0` (child), `> 0` (parent; value = child PID)
- **execvp()** on success: never returns — replaces the entire process memory image
- **execvp()** on failure: returns `-1`, sets `errno`
- **argv array for exec must be null-terminated** — last element = `NULL`
- **Zombie** = child exited, parent not yet waited. **Orphan** = parent exited, child re-parented to init
- **waitpid(-1, &s, WNOHANG)** in a loop = non-blocking zombie cleanup for any child

### Signals
- **SIGINT** = Ctrl+C; default: terminate process
- **SIGKILL** = cannot be caught or ignored
- **sigaction()** > `signal()` — more portable, doesn't auto-reset
- **Signal handler must only use async-signal-safe functions** — `write()` ✅ `printf()` ❌
- **EINTR:** syscall interrupted by signal → not an error → retry the call
- **Pre-compute signal handler strings** at startup with `snprintf()`; use only `write()` in handler

### Scheduling
- **FCFS problem:** convoy effect — long job blocks all shorter jobs
- **SJF:** optimal average wait (non-preemptive); needs burst time knowledge
- **SRTF:** preemptive SJF; preempts when shorter job arrives; globally optimal
- **RR quantum too small** = context switch overhead dominates; too large = becomes FCFS
- **Priority scheduling problem = starvation** of low-priority processes
- **Turnaround** = completion - arrival; **Waiting** = turnaround - burst

### Memory Management
- **Program break** = end of heap; move it with `sbrk(n)` which returns previous break
- **Don't call sbrk() frequently** — syscall overhead; get a large chunk and manage it
- **First-fit:** fastest (stop at first fit); **Best-fit:** least waste; **Worst-fit:** leaves biggest remainder
- **External fragmentation:** total free space sufficient but no single block big enough
- **Coalescing:** merging adjacent free blocks to combat fragmentation
- **strncpy()** does NOT guarantee null termination if source is too long → manually add `buf[n] = '\0'`
- **Free list header** stored in-place inside the free block itself (no separate malloc needed)

### Virtual Memory
- **Virtual address ≠ physical address** — translation done by MMU + page table
- **Paging:** fixed-size pages; no external fragmentation
- **Segmentation:** variable-size meaningful segments; suffers external fragmentation
- **VPN = VA / page_size; Offset = VA % page_size**
- **Page fault:** accessing unmapped page → kernel handles → transparent to process
- **Demand paging works** because of locality — programs don't need all pages at once
- **User processes only see virtual addresses** — kernel can see both

### Threads
- **Threads share:** text, data, BSS, heap; **Don't share:** stack (each thread has own)
- **Thread creation is lighter** than fork() — no new address space
- **`cnt++` is NOT atomic** — load, add, store (3 steps); data race in multi-thread context
- **Data race** = threads access shared data without synchronization → non-deterministic output
- **pthread_create returns 0 on success**, non-zero on error (unlike most syscalls)
- **Compile pthreads with `-pthread`** flag

---

## 🗂️ Topic → Lecture Cross-Reference

| Topic | Lecture | Lab |
|-------|---------|-----|
| OS layers, hardware, kernel/user mode | L1 | — |
| Memory hierarchy, von Neumann, ISA | L1 | — |
| Man pages, sleep(), pointers | L2.1 | — |
| fork(), exec(), wait(), errno | L2.2 | Lab 2 |
| getline(), strtok_r() | L2.1/L2.2 | Lab 1, Lab 3 |
| Function pointers, signals, sigaction | L3 | — |
| SIGINT, signal safety, EINTR | L3 | — |
| FCFS, SJF, SRTF, RR, Priority | L4 | — |
| Multilevel Queue, scheduling criteria | L4 | — |
| sbrk(), program break, heap layout | L5 | Lab 4 |
| Free list, first/best/worst fit | L5 | Lab 5 |
| Coalescing, fragmentation | L5 | Lab 5 |
| Virtual address space, paging | L6 | — |
| Segmentation, demand paging, page fault | L6 | — |
| Address translation, page table math | L6 | — |
| Threads, pthread, data race | L7 | — |

---

*Midterm Study Notes — CMPT 201 Spring 2026*
*Cross-referenced with sample midterm, lecture notes, and labs 1–5*
