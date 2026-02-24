# CMPT 201 — Practice Midterm 1
### Spring 2026 | Covers: Lectures 1–6, Brief Lecture 7

> **Format:** 5 MCQs · 10 Short Answer · 4 Long Answer/Coding
> **Approximate Time:** 90–105 minutes
> **Instructions:** Closed-book. Answer in the space provided. Keep short answers to 2–3 sentences. Verbose answers will result in point reductions.

---

## Section 1 — Multiple Choice (2 pts each = 10 pts)

*Circle the best answer.*

---

**Q1.** Which of the following statements best describes the difference between **kernel mode** and **user mode**?

- A) User mode allows direct hardware access; kernel mode is restricted to memory-safe operations only
- B) Kernel mode ("Ring 0") has full hardware privileges; user mode cannot execute privileged instructions or access kernel memory
- C) Kernel mode is used by regular user processes and kernel mode by the superuser (root)
- D) Both modes have the same privilege level; the difference is only in available system libraries

<details><summary>▶ Answer</summary>

**B** — Kernel mode has full privilege (Ring 0) and can directly access hardware and all memory. User mode is restricted — it cannot run privileged instructions or access kernel memory regions. This is unrelated to normal vs superuser — a root user is still running in user mode.

</details>

---

**Q2.** You call `fork()` and the return value is `0`. What does this mean for the executing code?

- A) `fork()` failed; the child was not created; `errno` is set
- B) The parent process is currently executing; `0` is a placeholder until the child's PID is assigned
- C) The child process is executing; in the parent, `fork()` returns the child's PID
- D) The process must call `exec()` immediately, otherwise behavior is undefined

<details><summary>▶ Answer</summary>

**C** — `fork()` returns `0` to the **child** process, and the **child's PID** to the parent. A return of `-1` indicates failure.

</details>

---

**Q3.** Consider the memory hierarchy from fastest/smallest to slowest/largest. Which ordering is correct?

- A) Registers → Cache → RAM → SSD → HDD → Tape
- B) Cache → Registers → RAM → HDD → SSD → Tape
- C) RAM → Cache → Registers → SSD → HDD → Tape
- D) Registers → RAM → Cache → SSD → HDD → Tape

<details><summary>▶ Answer</summary>

**A** — Registers (in-CPU, fastest, smallest) → L1/L2/L3 Cache → RAM (main memory) → SSD → HDD (spinning disk) → Tape (archival, slowest, largest).

</details>

---

**Q4.** Which function call is the correct way to install a signal handler for `SIGINT` in POSIX C?

- A) `signal(SIGINT, handler)` — preferred because it is simpler and always portable
- B) `kill(SIGINT, handler)` — sends the signal and sets the handler simultaneously
- C) `raise(SIGINT)` — registers the handler and triggers the signal for testing
- D) `sigaction(SIGINT, &sa, NULL)` — preferred because it is reliable and gives control over flags and masks

<details><summary>▶ Answer</summary>

**D** — `sigaction()` is the POSIX-preferred method. `signal()` has undefined behavior in some implementations (e.g., may reset to default after first delivery). `kill()` sends signals; `raise()` sends a signal to self.

</details>

---

**Q5.** In a memory allocator using a **free-list linked list**, you need to serve a request for 7 bytes. The free list has blocks of sizes: 6, 12, 24, 8, 4 (in order). What block does **best-fit** select?

- A) Block of size 6 (first block that's close enough)
- B) Block of size 12 (first block that fits)
- C) Block of size 24 (largest available)
- D) Block of size 8 (smallest block that is ≥ 7)

<details><summary>▶ Answer</summary>

**D** — Best-fit selects the **smallest block that is large enough** to satisfy the request. Block size 8 is the smallest block ≥ 7. Block 6 is too small; block 12 fits but wastes more space; block 8 wastes the least.

</details>

---

## Section 2 — Short Answer (2 pts each = 20 pts)

*Answer in 2–3 sentences maximum. Use only the space provided.*

---

**Q6.** (2 pts) What is the difference between a **program** and a **process**?

```
Your answer:




```

<details><summary>▶ Answer</summary>

A **program** is a compiled executable file stored on disk — it is static and only runs when invoked. A **process** is a running instance of a program; the OS creates a memory space for it, loads the code into memory, and the CPU begins executing it. Multiple processes can be created from the same program file simultaneously.

</details>

---

**Q7.** (2 pts) What is a **zombie process**? When does one appear, and why does the OS keep the zombie entry in the process table?

```
Your answer:




```

<details><summary>▶ Answer</summary>

A zombie process is a child process that has **finished executing** but whose exit status hasn't been collected yet by the parent. It appears when a child calls `exit()` before the parent calls `wait()`/`waitpid()`. The OS keeps the entry in the process table because the exit status must be preserved so the parent can retrieve it — once `waitpid()` is called, the entry is removed.

</details>

---

**Q8.** (2 pts) What is the role of the **kernel** in an operating system? Name two of its core responsibilities.

```
Your answer:




```

<details><summary>▶ Answer</summary>

The kernel is the central part of the OS that actively manages computer resources. Two core responsibilities: (1) **Resource management** — mediating access to hardware (CPU, memory, I/O) among competing processes, and (2) **Process control** — starting, stopping, and scheduling processes. It also provides **protection** — isolating processes from each other and from the kernel itself.

</details>

---

**Q9.** (2 pts) Explain what happens when `execvp()` is called and it **succeeds**. What happens to the calling process?

```
Your answer:




```

<details><summary>▶ Answer</summary>

When `execvp()` succeeds, the calling process's entire memory image (code, data, stack, heap) is **replaced** by the new program being executed. The new program starts from its own `main()`. The original calling process's code after `execvp()` is never reached — the function only returns if it **fails**.

</details>

---

**Q10.** (2 pts) Why is `printf()` **not** safe to use inside a signal handler? What should be used instead?

```
Your answer:




```

<details><summary>▶ Answer</summary>

`printf()` uses an internal buffer protected by a mutex lock. If a signal interrupts `printf()` while it holds the lock and the handler also calls `printf()`, a **deadlock** can occur, or the internal buffer state can be corrupted. The async-signal-safe alternative is `write()`, which is a direct syscall with no internal state that can be corrupted.

</details>

---

**Q11.** (2 pts) What is **preemptive scheduling**? Give one example of a scheduling algorithm that is preemptive.

```
Your answer:




```

<details><summary>▶ Answer</summary>

Preemptive scheduling means the scheduler can **interrupt a currently running process** and context-switch to another process before the running process finishes or voluntarily yields. One example is **Round Robin** — each process gets a fixed time quantum and is preempted when it expires, returning to the ready queue.

</details>

---

**Q12.** (2 pts) What is the **program break** in relation to heap memory, and what does `sbrk()` do?

```
Your answer:




```

<details><summary>▶ Answer</summary>

The program break is the boundary marking the **end of the heap** — specifically, the first address beyond the end of the uninitialized data (BSS) segment. `sbrk(n)` moves the program break upward by `n` bytes, effectively expanding the heap and returning the previous break address (which is the start of the newly allocated region).

</details>

---

**Q13.** (2 pts) Describe what **virtual memory** is and name its two main components as discussed in lecture.

```
Your answer:




```

<details><summary>▶ Answer</summary>

Virtual memory is an OS abstraction that gives each process the **illusion of having its own entire address space**, enabling physical memory sharing and process isolation. Its two main components are: (1) the **virtual address space** — the abstraction giving each process addresses from 0 to max, and (2) **address translation** — the mechanism (via page table + hardware) that maps virtual addresses to physical addresses.

</details>

---

**Q14.** (2 pts) What are the three main **Section man page** numbers used in this course, and what does each cover?

```
Your answer:




```

<details><summary>▶ Answer</summary>

- `man 1` — **General commands** (shell commands, e.g., `man 1 ls`)
- `man 2` — **System calls** (kernel-level calls, e.g., `man 2 fork`)
- `man 3` — **C standard library functions** (libc functions, e.g., `man 3 printf`)

</details>

---

**Q15.** (2 pts) Why does Moore's Law have two "phases"? What changed around 2005?

```
Your answer:




```

<details><summary>▶ Answer</summary>

The original Moore's Law described the doubling of transistors on a chip, which translated to faster **single-core** clock speeds. Around 2005, physical limits (heat, power) made further single-core speedup impractical, so chip designers shifted focus to adding **multiple cores** — running multiple instruction sequences simultaneously rather than making one sequence faster.

</details>

---

## Section 3 — Long Answer / Coding (4 pts each = 16 pts)

---

**Q16.** (4 pts) Write a complete C program that does the following:
- Creates a **child process** using `fork()`
- The **child** uses `execvp()` to run `ls -a -l`
- The **parent** uses `waitpid()` to wait for the **specific** child to finish
- If the child exited normally (use `WIFEXITED`), the parent prints `"Child done."`
- Handle `fork()` failure by printing an error and calling `exit(1)`

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {

    // Write your solution here:

}
```

<details><summary>▶ Answer</summary>

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        exit(1);
    } else if (pid == 0) {
        // Child process
        char *args[] = {"ls", "-a", "-l", NULL};
        execvp("ls", args);
        // If execvp returns, it failed:
        perror("execvp");
        exit(EXIT_FAILURE);
    } else {
        // Parent process
        int wstatus;
        waitpid(pid, &wstatus, 0);   // wait for THIS specific child
        if (WIFEXITED(wstatus)) {
            printf("Child done.\n");
        }
    }
    return 0;
}
```

**Key points:** `execvp()` arg array must be null-terminated. `waitpid(pid, ...)` not `waitpid(-1, ...)`. `WIFEXITED` checks normal exit.

</details>

---

**Q17.** (4 pts) The following code has a **signal handler bug** and **one missing statement** in `main()`. Identify and fix both issues. Write the corrected code.

```c
#define _POSIX_C_SOURCE 200809L
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

void handle_sigint(int signum) {
    printf("Caught signal %d\n", signum);
}

int main(void) {
    struct sigaction sa;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);
    // MISSING: ??
    sigaction(SIGINT, &sa, NULL);
    while (1) {
        sleep(1);
    }
    return 0;
}
```

**Bug 1 (in handler):** _______________________________________________

**Bug 2 (missing in main):** _______________________________________________

**Corrected code:**

```c

```

<details><summary>▶ Answer</summary>

**Bug 1:** `printf()` is not async-signal-safe. Inside a signal handler, use `write()` instead.

**Bug 2:** The handler function pointer is never assigned to `sa.sa_handler`. Missing: `sa.sa_handler = handle_sigint;`

```c
#define _POSIX_C_SOURCE 200809L
#include <signal.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>

void handle_sigint(int signum) {
    // FIX 1: Use write() not printf()
    const char *msg = "Caught SIGINT\n";
    write(STDOUT_FILENO, msg, strlen(msg));
}

int main(void) {
    struct sigaction sa;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);
    sa.sa_handler = handle_sigint;   // FIX 2: assign handler
    sigaction(SIGINT, &sa, NULL);
    while (1) {
        sleep(1);
    }
    return 0;
}
```

</details>

---

**Q18.** (4 pts) Consider the following scheduling scenario using **Shortest Remaining Time First (SRTF)**. Three processes arrive at the times shown:

| Process | Burst Time | Arrival Time |
|---------|-----------|--------------|
| P1      | 8 ms      | 0 ms         |
| P2      | 4 ms      | 2 ms         |
| P3      | 2 ms      | 4 ms         |

**(a) (2 pts)** Draw the execution timeline (Gantt chart style).

```
Time: 0    2    4    6    7    11
      |____|____|____|____|____|
      [    ][   ][   ][   ][   ]
```

**(b) (2 pts)** Calculate the **average waiting time** for all three processes.

```
P1 waiting time: ______
P2 waiting time: ______
P3 waiting time: ______
Average: ______
```

<details><summary>▶ Answer</summary>

**SRTF** preempts current process when a new one arrives with shorter remaining time.

**Execution trace:**
- t=0: P1 starts (remaining: 8)
- t=2: P2 arrives (remaining: 4). P1 remaining=6. P2 shorter → **preempt P1, run P2**
- t=4: P3 arrives (remaining: 2). P2 remaining=2. Same → **preempt P2, run P3**
- t=6: P3 finishes. P2 remaining=2, P1 remaining=6. Run P2.
- t=8: P2 finishes. Run P1 (remaining=6).
- t=14: P1 finishes.

**Gantt chart:**
```
[P1: 0-2][P2: 2-4][P3: 4-6][P2: 6-8][P1: 8-14]
```

**Waiting times:**
- P1: started at 0, total burst = 8, finished at 14. Waiting = 14 - 0 - 8 = **6 ms**
- P2: total burst = 4, ran 2-4 and 6-8, finished at 8. Waiting = 8 - 2 - 4 = **2 ms**
- P3: arrived at 4, ran 4-6, no wait. Waiting = **0 ms**
- **Average = (6 + 2 + 0) / 3 = 2.67 ms**

</details>

---

**Q19.** (4 pts) **Fill in the missing code** below. The program should use `sbrk()` to allocate 128 bytes of heap space, treat it as a `struct header` block, set its `size` to 128 and `next` to `NULL`, and then print both fields using `write()`. Fill in the six blanked sections.

```c
#define _POSIX_C_SOURCE 200809L
#include <stdint.h>
#include <unistd.h>
#include <string.h>

struct header {
    uint64_t size;
    struct header *next;
};

int main(void) {
    // 1. Allocate 128 bytes on the heap using sbrk()
    struct header *block = ______________;

    if (block == (void *)-1) {
        // sbrk failed
        return 1;
    }

    // 2. Set the size field to 128
    block->________ = ________;

    // 3. Set next to NULL
    block->next = ________;

    // 4. Build and write the size output (no printf allowed)
    char buf[64];
    int len = snprintf(buf, sizeof(buf), "size: %lu\n",
                       (unsigned long) block->________);
    write(________, buf, len);

    return 0;
}
```

<details><summary>▶ Answer</summary>

```c
// 1.
struct header *block = (struct header *) sbrk(128);

// 2.
block->size = 128;

// 3.
block->next = NULL;

// 4.
(unsigned long) block->size
write(STDOUT_FILENO, buf, len);
```

**Key point:** `sbrk(n)` returns the *previous* break (start of new memory) as `void*`. Cast to your struct pointer. `STDOUT_FILENO` = file descriptor 1.

</details>

---

## Answer Summary

| Q | Pts | Topic |
|---|-----|-------|
| 1 | 2 | Kernel/User mode |
| 2 | 2 | fork() return values |
| 3 | 2 | Memory hierarchy |
| 4 | 2 | sigaction() |
| 5 | 2 | Best-fit allocation |
| 6 | 2 | Program vs Process |
| 7 | 2 | Zombie processes |
| 8 | 2 | Kernel roles |
| 9 | 2 | execvp() |
| 10 | 2 | Signal safety |
| 11 | 2 | Preemptive scheduling |
| 12 | 2 | sbrk() / program break |
| 13 | 2 | Virtual memory |
| 14 | 2 | Man page sections |
| 15 | 2 | Moore's Law |
| 16 | 4 | fork/exec/waitpid coding |
| 17 | 4 | Signal handler bugs |
| 18 | 4 | SRTF scheduling |
| 19 | 4 | sbrk() + struct fill-in |
| **Total** | **46** | |

---
*CMPT 201 Practice Midterm 1 — Spring 2026*
