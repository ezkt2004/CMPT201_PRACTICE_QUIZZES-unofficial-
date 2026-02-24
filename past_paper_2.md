# CMPT 201 — Practice Midterm 2
### Spring 2026 | Covers: Lectures 1–6, Brief Lecture 7

> **Format:** 5 MCQs · 10 Short Answer · 4 Long Answer/Coding
> **Approximate Time:** 90–105 minutes
> **Instructions:** Closed-book. Answer in the space provided. Keep short answers to 2–3 sentences. Verbose answers will result in point reductions.

---

## Section 1 — Multiple Choice (2 pts each = 10 pts)

*Circle the best answer.*

---

**Q1.** What is the **primary benefit** of virtual memory for user processes?

- A) It allows user processes to directly access physical memory addresses for maximum performance
- B) It eliminates the need for a page table by making all memory accesses the same speed
- C) Each process gets its own isolated virtual address space, enabling memory isolation and the illusion of having the entire address range
- D) It compresses physical memory so that more programs can be loaded simultaneously

<details><summary>▶ Answer</summary>

**C** — Virtual memory gives each process its own virtual address space (0 to address-max), providing **memory isolation** (a process cannot see another's memory) and the illusion of having the whole address space. A user process *only* uses virtual addresses, never physical ones.

</details>

---

**Q2.** Which of the following correctly describes what happens when a signal interrupts a blocking `read()` call?

- A) `read()` panics and the process terminates with `SIGSEGV`
- B) `read()` returns `-1` with `errno` set to `EINTR`; this is not a real error — the call should be retried
- C) `read()` returns `0` indicating end-of-file; the signal was consumed
- D) The signal is queued until `read()` finishes; `read()` always completes before the handler runs

<details><summary>▶ Answer</summary>

**B** — When a signal arrives and interrupts a blocking syscall like `read()`, the call returns `-1` with `errno == EINTR` (Interrupted system call). This is **not a true error** — the correct response is to check for `EINTR` and restart the call (e.g., `continue` in a loop).

</details>

---

**Q3.** You have a virtual address space of 64 KB managed with paging, and the page size is 4 KB. How many bits are needed for the **page offset**, and how many **entries** are in the page table?

- A) 10 offset bits, 64 entries
- B) 12 offset bits, 16 entries
- C) 12 offset bits, 64 entries
- D) 10 offset bits, 16 entries

<details><summary>▶ Answer</summary>

**B** — Page size = 4 KB = 4096 = 2^12 → **12 offset bits**. Number of pages = 64 KB / 4 KB = 16 → **16 page table entries**.

</details>

---

**Q4.** In the following `strtok_r()` usage, what is wrong?

```c
char *saveptr;
token = strtok_r(input, " ", &saveptr);
while (token != NULL) {
    token = strtok_r(input, " ", &saveptr); // subsequent calls
}
```

- A) `strtok_r` cannot use `" "` (space) as a delimiter; only `"\n"` is valid
- B) `saveptr` must be initialized to `NULL` before the first call
- C) Subsequent calls must pass `NULL` (not `input`) as the first argument; passing `input` restarts tokenization from the beginning each iteration
- D) `strtok_r` modifies `saveptr` after the first call making it invalid for subsequent calls

<details><summary>▶ Answer</summary>

**C** — For `strtok_r()`, the **first call** passes the string; **all subsequent calls** must pass `NULL` as the first argument. The function uses `saveptr` internally to track progress. Passing `input` again on each iteration restarts from the beginning, creating an infinite loop.

</details>

---

**Q5.** Which statement best describes an **orphan process**?

- A) A process that has called `exec()` but `exec()` has not yet returned
- B) A child process whose parent exits first, causing the child to be adopted by `init` (PID 1)
- C) A process that has exited but whose parent hasn't called `wait()`, leaving it in the process table
- D) A process blocked in the I/O queue that can never return to the ready queue

<details><summary>▶ Answer</summary>

**B** — An **orphan** is a child whose **parent exits first** while the child is still running. The OS re-parents the orphan to `init` (PID 1), which will call `wait()` on it when it eventually finishes. A zombie (option C) is different — the child exits first while the parent is still running but hasn't called `wait()`.

</details>

---

## Section 2 — Short Answer (2 pts each = 20 pts)

*Answer in 2–3 sentences maximum.*

---

**Q6.** (2 pts) What is a **context switch**? Why does it have overhead?

```
Your answer:




```

<details><summary>▶ Answer</summary>

A context switch is the act of **stopping one process and starting/resuming another** on the CPU. It has overhead because the OS must save the full state of the current process (registers, program counter, memory mappings) and restore the state of the next process — this involves many complex steps and memory operations. Frequent context switching wastes CPU time on management rather than actual computation.

</details>

---

**Q7.** (2 pts) Explain the difference between **segmentation** and **paging** as approaches to virtual memory address translation.

```
Your answer:




```

<details><summary>▶ Answer</summary>

**Paging** divides the virtual address space into **fixed-size** regions called pages (e.g., 4 KB), mapped to equally-sized physical frames. **Segmentation** divides the space into **variable-size**, semantically meaningful regions (text, stack, heap, data). Paging avoids external fragmentation; segmentation aligns with the programmer's view of memory but can suffer from external fragmentation.

</details>

---

**Q8.** (2 pts) What is a **function pointer** in C? Give one reason why signal handlers require function pointers.

```
Your answer:




```

<details><summary>▶ Answer</summary>

A function pointer is a variable that **holds the memory address of a function**, allowing functions to be passed as arguments, stored in structs, or called indirectly. Signal handlers require function pointers because the kernel needs a way to call *your* handler code when a signal arrives — you register your handler's address with `sigaction()` via the `sa_handler` field, which is a function pointer.

</details>

---

**Q9.** (2 pts) What is the difference between **32-bit and 64-bit architectures**? What is the practical impact on pointer size and maximum addressable memory?

```
Your answer:




```

<details><summary>▶ Answer</summary>

The difference is in the CPU's **instruction set** and the width of its registers and memory bus. In a 32-bit architecture, pointers are 4 bytes and the max addressable memory is 2^32 = ~4 GB. In 64-bit, pointers are 8 bytes and the theoretical max is 2^64 addresses — far more than current physical RAM, which is why virtual memory spans such a vast range.

</details>

---

**Q10.** (2 pts) Describe **demand paging**. Why does it work well in practice?

```
Your answer:




```

<details><summary>▶ Answer</summary>

Demand paging is a strategy where a page is brought into physical memory **only when it is actually accessed** (on demand), rather than loading the entire program upfront. It works well because of **temporal and spatial locality** — a typical program only accesses a small portion of its memory at any given time, so most pages don't need to be loaded at all during a short execution window.

</details>

---

**Q11.** (2 pts) Why do we use `sbrk()` sparingly (not call it for every small allocation)?  What do we do instead?

```
Your answer:




```

<details><summary>▶ Answer</summary>

`sbrk()` is a **syscall that crosses the user-kernel boundary**, which is expensive in terms of overhead. It is better to call `sbrk()` once to get a large chunk of memory, then manage that chunk piece-by-piece in user space (using a memory allocator with a free-list) to serve many small `malloc()` requests without repeated syscalls.

</details>

---

**Q12.** (2 pts) List the data structures (segments) in a typical virtual address space layout, from lowest to highest address.

```
Your answer:




```

<details><summary>▶ Answer</summary>

From lowest to highest: **Text** (code/instructions) → **Data** (initialized global/static variables) → **BSS** (uninitialized globals) → **Heap** (grows upward, dynamic allocation) → [gap / memory mapping region] → **Stack** (grows downward, local variables, function calls) → **Kernel** (top of address space, inaccessible from user mode).

</details>

---

**Q13.** (2 pts) Why is the kernel described as **event-driven**? Give two examples of events it responds to.

```
Your answer:




```

<details><summary>▶ Answer</summary>

The kernel is event-driven because it does not run continuously — it responds to **events** that occur. Examples of events include: (1) a **hardware interrupt** (e.g., keyboard press, mouse click, timer expiry), (2) a **system call** from a user process (e.g., `write()`, `fork()`), and (3) a **signal** (e.g., `SIGINT` from Ctrl+C, `SIGSEGV` from an invalid memory access).

</details>

---

**Q14.** (2 pts) What is the **starvation** problem in Priority Scheduling, and how does **Multilevel Queue Scheduling** address it?

```
Your answer:




```

<details><summary>▶ Answer</summary>

Starvation occurs when low-priority processes **never get CPU time** because high-priority processes keep arriving and preempting them. Multilevel Queue Scheduling mitigates this by grouping processes into separate queues with different priorities and using a scheme like **weighted Round Robin** across queues — giving each queue a guaranteed share of CPU time rather than completely blocking lower-priority queues.

</details>

---

**Q15.** (2 pts) What is the **von Neumann architecture**? Identify its two fundamental components.

```
Your answer:




```

<details><summary>▶ Answer</summary>

The von Neumann architecture is the fundamental design model for modern computers, where a CPU **fetches data from memory** to provide it for computation. Its two fundamental components are: (1) **Computation** — handled by the CPU (processing instructions), and (2) **Data** — handled by memory (RAM and storage, storing data and programs).

</details>

---

## Section 3 — Long Answer / Coding (4 pts each = 16 pts)

---

**Q16.** (4 pts) Write a **complete C program** that:
- Reads a line of user input using `getline()` (dynamically allocates buffer)
- Tokenizes it by spaces using `strtok_r()`
- Prints each token on its own line prefixed by its index (0-based), e.g. `"0: Hello"`
- Correctly **frees** the `getline()` buffer before exiting
- Handles `getline()` failure (returns -1)

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    // Write your solution here:

}
```

<details><summary>▶ Answer</summary>

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    char *buf = NULL;
    size_t buf_size = 0;

    printf("Enter text: ");
    fflush(stdout);

    ssize_t n = getline(&buf, &buf_size, stdin);
    if (n == -1) {
        perror("getline");
        free(buf);
        return 1;
    }

    char *saveptr;
    char *token = strtok_r(buf, " \n", &saveptr);
    int i = 0;
    while (token != NULL) {
        printf("%d: %s\n", i++, token);
        token = strtok_r(NULL, " \n", &saveptr); // NULL on subsequent calls!
    }

    free(buf);  // getline dynamically allocates — must free
    return 0;
}
```

**Key points:** `buf = NULL` and `buf_size = 0` tells `getline()` to allocate. `strtok_r()` subsequent calls use `NULL`. Always `free(buf)`.

</details>

---

**Q17.** (4 pts) Fill in the **three blanked sections** of the code below. The program creates two processes using fork(). The **parent** prints its own PID. The **child** sleeps for 2 seconds then prints its PID and parent's PID. Both handle `fork()` failure.

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    pid_t pid = fork();

    if (pid == ________) {
        // fork failed
        perror("fork");
        exit(1);
    }

    if (pid > 0) {
        // Parent
        printf("Parent PID: %d\n", ________);

    } else {
        // Child
        sleep(2);
        printf("Child PID: %d, Parent PID: %d\n",
               ________, ________);
    }

    return 0;
}
```

Fill in the four blanks:
1. `pid == ________`
2. `getpid()` call for parent: `________`
3. `getpid()` call for child: `________`
4. `getppid()` call for child: `________`

<details><summary>▶ Answer</summary>

```c
if (pid == -1) {          // blank 1: -1 means fork failed

printf("Parent PID: %d\n", getpid());   // blank 2: getpid()

printf("Child PID: %d, Parent PID: %d\n",
       getpid(),           // blank 3: child's own PID
       getppid());         // blank 4: child's parent PID
```

**`getpid()`** = current process's PID. **`getppid()`** = parent's PID. **`fork()` fails** when return is `-1`.

</details>

---

**Q18.** (4 pts) Consider the following **virtual address** scenario. A system uses paging with:
- Virtual address space: 256 KB (18-bit addresses)
- Page size: 8 KB (8192 bytes)

**(a) (1 pt)** How many bits are used for the **page offset**?

```
Answer: _____
```

**(b) (1 pt)** How many bits are used for the **virtual page number (VPN)**?

```
Answer: _____
```

**(c) (1 pt)** How many **entries** are in the page table?

```
Answer: _____
```

**(d) (1 pt)** A virtual address is `0x0A000` (in hex). What is its **VPN** and **offset** (in decimal)?

> Hint: 8 KB = 8192. Convert `0x0A000` to decimal = 40960.

```
VPN = _____      Offset = _____
```

<details><summary>▶ Answer</summary>

**(a)** Page size = 8 KB = 8192 = 2^13 → **13 offset bits**

**(b)** Total bits = 18 (for 256 KB = 2^18). VPN bits = 18 - 13 = **5 bits**

**(c)** 2^5 = **32 entries** in the page table

**(d)** Virtual address = 40960 decimal.
- VPN = 40960 / 8192 = **5**
- Offset = 40960 % 8192 = 40960 - (5 × 8192) = 40960 - 40960 = **0**

So VPN = 5, Offset = 0.

</details>

---

**Q19.** (4 pts) The function below is supposed to implement **first-fit allocation** — traverse a free-list linked list and return the `id` of the first block with sufficient size. Complete the function.

```c
struct header {
    uint64_t size;
    struct header *next;
    int id;
};

// Returns the id of the first block >= requested size.
// Returns -1 if no suitable block found.
int find_first_fit(struct header *free_list_ptr, uint64_t size) {

    struct header *current = ____________________;

    while (current ____________) {

        if (current->size ____________ size) {
            return ____________;
        }

        current = ____________;
    }

    return -1;
}
```

Fill in the five blanks, then answer: **What would best-fit change about this function?** (1–2 sentences)

```
Best-fit change:



```

<details><summary>▶ Answer</summary>

```c
int find_first_fit(struct header *free_list_ptr, uint64_t size) {
    struct header *current = free_list_ptr;   // blank 1: start at head

    while (current != NULL) {                 // blank 2: loop condition

        if (current->size >= size) {          // blank 3: big enough?
            return current->id;               // blank 4: return its id
        }

        current = current->next;              // blank 5: advance pointer
    }

    return -1;
}
```

**Best-fit change:** Instead of returning the **first** block that fits, best-fit would iterate through the **entire list**, tracking the block with the **smallest size that is still ≥ requested size**, and returning that block's id at the end.

</details>

---

## Answer Summary

| Q | Pts | Topic |
|---|-----|-------|
| 1 | 2 | Virtual memory purpose |
| 2 | 2 | EINTR / signal + read() |
| 3 | 2 | Paging arithmetic |
| 4 | 2 | strtok_r() usage |
| 5 | 2 | Orphan process |
| 6 | 2 | Context switch |
| 7 | 2 | Paging vs Segmentation |
| 8 | 2 | Function pointers |
| 9 | 2 | 32 vs 64-bit |
| 10 | 2 | Demand paging |
| 11 | 2 | sbrk() cost |
| 12 | 2 | Address space layout |
| 13 | 2 | Event-driven kernel |
| 14 | 2 | Priority scheduling / starvation |
| 15 | 2 | Von Neumann architecture |
| 16 | 4 | getline() + strtok_r() coding |
| 17 | 4 | fork() fill-in |
| 18 | 4 | Paging calculation |
| 19 | 4 | First-fit linked list |
| **Total** | **46** | |

---
*CMPT 201 Practice Midterm 2 — Spring 2026*
