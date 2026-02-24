# CMPT 201 — Practice Midterm 3
### Spring 2026 | Covers: Lectures 1–6, Brief Lecture 7

> **Format:** 5 MCQs · 10 Short Answer · 4 Long Answer/Coding
> **Approximate Time:** 90–105 minutes
> **Instructions:** Closed-book. Answer in the space provided. Keep short answers to 2–3 sentences. Verbose answers will result in point reductions.

---

## Section 1 — Multiple Choice (2 pts each = 10 pts)

*Circle the best answer.*

---

**Q1.** What does the second argument to `sigaction()` represent?

- A) The signal number to respond to (e.g., `SIGINT`)
- B) A mask of signals to block during handler execution
- C) A function pointer directly to the signal handler
- D) A pointer to a `struct sigaction` which contains a function pointer to the handler

<details><summary>▶ Answer</summary>

**D** — The second argument is `const struct sigaction *act`, a pointer to a `struct sigaction`. The struct's `sa_handler` field is the function pointer to your handler. You cannot pass the function pointer directly to `sigaction()`.

</details>

---

**Q2.** Consider this code: `pid_t pid = fork();`. Which of following statements is **always true** immediately after this line, if `fork()` succeeds?

- A) There is one process; `pid` holds the PID of the current process
- B) There are two copies of the program running; `pid == 0` in the child, `pid > 0` in the parent
- C) The child immediately calls `exec()` and the parent waits
- D) The parent continues but the child is in the I/O queue waiting for resources

<details><summary>▶ Answer</summary>

**B** — After a successful `fork()`, **two identical copies** of the program exist and run concurrently. In the **child**, `fork()` returns `0`. In the **parent**, `fork()` returns the child's PID (a positive integer). No automatic `exec()` or waiting occurs.

</details>

---

**Q3.** In the **Round Robin** scheduling algorithm, what is a **quantum**, and what is the effect of making it extremely small?

- A) A quantum determines process priority; smaller quanta run higher-priority processes more often
- B) A quantum is the time slice each process gets; extremely small quanta cause excessive context switch overhead, leaving little time for actual work
- C) A quantum is the amount of memory allocated per process; smaller quanta cause more page faults
- D) A quantum is the number of I/O operations a process can make; smaller quanta reduce I/O starvation

<details><summary>▶ Answer</summary>

**B** — In Round Robin, a **quantum** is the fixed time slice each process gets before being preempted. If the quantum is too small, the CPU spends *most* of its time doing context switches rather than running actual process code — the overhead of switching dominates useful work.

</details>

---

**Q4.** What is `errno` and when is it useful?

- A) A global variable set by system calls and library functions when an error occurs; it identifies *which* error happened
- B) A function that prints the error code of the last syscall to `stderr`
- C) A macro that evaluates to `true` if the last syscall succeeded and `false` otherwise
- D) A return code from `fork()` indicating the child failed to start

<details><summary>▶ Answer</summary>

**A** — `errno` is a **global integer variable** (defined in `errno.h`) that is set by system calls and library functions when an error occurs. The return value of the function tells you *that* something went wrong; `errno` tells you *what* went wrong (e.g., `EAGAIN`, `ENOMEM`, `ECHILD`, `EINTR`).

</details>

---

**Q5.** In a **data race**, two threads increment a shared counter `cnt` 10,000,000 times each, expecting a final value of 20,000,000. The actual output is often less. Why?

- A) The OS limits each thread to a maximum of 5,000,000 increments for fairness
- B) `cnt++` compiles to a single atomic instruction so the result should always be 20,000,000
- C) `cnt++` is not atomic — it involves load, add, and store steps; two threads can interleave these steps and overwrite each other's result
- D) Threads share the stack, causing the counters stored on the stack to corrupt each other

<details><summary>▶ Answer</summary>

**C** — `cnt++` in C compiles to **three non-atomic steps**: (1) load `cnt` from memory, (2) add 1, (3) store result back. If two threads both load the same value of `cnt` before either stores, both compute the same incremented value, and one update is **lost** — this is the data race. Threads do NOT share stacks (each has its own), but they share global/heap variables.

</details>

---

## Section 2 — Short Answer (2 pts each = 20 pts)

*Answer in 2–3 sentences maximum.*

---

**Q6.** (2 pts) What does `WIFEXITED(wstatus)` check, and when would you use it after `waitpid()`?

```
Your answer:




```

<details><summary>▶ Answer</summary>

`WIFEXITED(wstatus)` returns **true** if the child process terminated normally by calling `exit()` or returning from `main()`. You use it after `waitpid()` to determine whether the child exited cleanly versus being killed by a signal (check with `WIFSIGNALED`). If it returns true, you can use `WEXITSTATUS(wstatus)` to retrieve the exit code.

</details>

---

**Q7.** (2 pts) Explain **external fragmentation** in the context of a memory allocator. Which allocation strategy minimizes it: first-fit, best-fit, or worst-fit?

```
Your answer:




```

<details><summary>▶ Answer</summary>

External fragmentation occurs when the **total free memory is sufficient** for a request, but it is split into small non-contiguous blocks — no single block is large enough. There is no universally best strategy, but **best-fit** can reduce wasted space per allocation (smallest sufficient block), while **worst-fit** (largest block) leaves larger leftover fragments that may be reusable. First-fit is fast but can fragment the beginning of the list.

</details>

---

**Q8.** (2 pts) What does `getline()` do that `fgets()` does not? Why is memory management important when using `getline()`?

```
Your answer:




```

<details><summary>▶ Answer</summary>

`getline()` **dynamically allocates or grows** its buffer as needed to fit the entire line, whereas `fgets()` uses a fixed-size caller-provided buffer and truncates if the line is too long. Because `getline()` calls `malloc()` internally, the caller is responsible for calling `free()` on the buffer when done — failing to do so causes a **memory leak**.

</details>

---

**Q9.** (2 pts) Why does `exec()` never return on success? What happens to the process that called it?

```
Your answer:




```

<details><summary>▶ Answer</summary>

When `exec()` succeeds, the calling process's **entire memory image** is replaced — its code, data, heap, and stack are all overwritten by the new program. There is literally no code left from the original program to return to. The process continues running, but it is now executing the new program's `main()` function in the same process slot (same PID).

</details>

---

**Q10.** (2 pts) What is **coalescing** in a memory allocator, and why is it important?

```
Your answer:




```

<details><summary>▶ Answer</summary>

Coalescing is the process of **merging adjacent free blocks** into a single larger free block when a block is released. It is important because without coalescing, repeated allocation and deallocation creates many small fragmented free blocks that individually cannot satisfy larger requests, even though the combined free space could — this is external fragmentation. Coalescing combats this by rebuilding larger usable blocks.

</details>

---

**Q11.** (2 pts) Explain the difference between **turnaround time** and **waiting time** in CPU scheduling.

```
Your answer:




```

<details><summary>▶ Answer</summary>

**Turnaround time** is the total time from when a process is **submitted** (arrives) until it **completes** — it includes both the actual execution time and any waiting time. **Waiting time** is the time a process spends in the **ready queue** waiting to run (not counting I/O waits or actual CPU time). Turnaround = waiting time + burst time.

</details>

---

**Q12.** (2 pts) In the context of virtual memory, what happens when a process accesses a page that is **not currently in physical memory**?

```
Your answer:




```

<details><summary>▶ Answer</summary>

This triggers a **page fault** — the MMU hardware detects that the virtual page is not mapped to a physical frame and interrupts the process. The kernel's page fault handler then retrieves the page from disk (swap space or file system), loads it into a free physical frame, updates the page table, and **resumes the process** from the faulting instruction as if nothing happened.

</details>

---

**Q13.** (2 pts) What are two concrete examples of **systems programming tasks** that require a systems programming language like C (as opposed to a high-level language like Python or Java)?

```
Your answer:




```

<details><summary>▶ Answer</summary>

Two examples: (1) **Writing an OS kernel or device driver** — requires direct hardware register access, raw memory manipulation, and precise control over memory layout, which Python/Java abstract away. (2) **Implementing a memory allocator** (like `malloc()`) — requires pointer arithmetic, direct management of raw memory addresses, and use of syscalls like `sbrk()` that are not exposed in high-level languages.

</details>

---

**Q14.** (2 pts) In **First Come First Served (FCFS) scheduling**, describe the convoy effect and why it is a problem.

```
Your answer:




```

<details><summary>▶ Answer</summary>

The convoy effect occurs in FCFS when a **long CPU-bound process** arrives first and monopolizes the CPU for a long time. All shorter processes that arrived later must wait behind it in the ready queue — even if they only need a short CPU burst. This significantly increases average waiting time and is the main weakness of FCFS, which **Shortest Job First (SJF) and Round Robin** are designed to address.

</details>

---

**Q15.** (2 pts) What memory segment is used for each of the following C variables? Assume they are in a regular function: (a) `int x = 5;` declared inside `main()`, (b) `static int count = 0;` declared inside a function, (c) `int *ptr = malloc(100);` — where does `ptr` itself live, and where does the allocated block live?

```
Your answer:




```

<details><summary>▶ Answer</summary>

(a) `int x = 5;` — local variable → stored on the **stack**
(b) `static int count = 0;` — initialized static → stored in the **Data segment** (persists across function calls)
(c) `ptr` itself is a local variable → stored on the **stack**; the 100-byte block pointed to by `ptr` is dynamically allocated → stored on the **heap**

</details>

---

## Section 3 — Long Answer / Coding (4 pts each = 16 pts)

---

**Q16.** (4 pts) **Code analysis — predict the output.**

Carefully trace through this code and write the exact output. Assume `fork()` always succeeds. Write "PARENT" or "CHILD" in brackets to clarify which process prints each line.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    printf("Before fork\n");
    fflush(stdout);

    pid_t pid = fork();

    if (pid == 0) {
        printf("In child, pid=%d\n", getpid());
        exit(0);
    } else {
        int wstatus;
        waitpid(pid, &wstatus, 0);
        if (WIFEXITED(wstatus)) {
            printf("Child exited with status %d\n", WEXITSTATUS(wstatus));
        }
        printf("In parent, done\n");
    }
    return 0;
}
```

**Write the exact output (3 lines):**

```
Line 1: ______________________________________
Line 2: ______________________________________
Line 3: ______________________________________
Line 4: ______________________________________
```

<details><summary>▶ Answer</summary>

```
Before fork
In child, pid=<some PID>     (printed by child)
Child exited with status 0   (printed by parent after waitpid)
In parent, done              (printed by parent)
```

**Trace:**
1. `"Before fork"` — single process prints this before `fork()`
2. After `fork()`, child gets `pid==0`, prints its own PID, calls `exit(0)`
3. Parent gets `pid>0`, calls `waitpid()` — blocks until child exits; child exits with code `0`
4. `WIFEXITED` is true, `WEXITSTATUS` = 0 → parent prints "Child exited with status 0"
5. Parent prints "In parent, done"

Note: the actual child PID number is non-deterministic but the output structure is always these 4 lines in this order.

</details>

---

**Q17.** (4 pts) **Fill in the missing code** to complete this program that creates a thread using `pthread_create()`. The thread receives a string argument, prints it, and returns the string's length. The main thread waits for the thread and prints the return value.

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <string.h>

// Thread function: receives a string, prints it, returns its length
void *thread_func(void *arg) {
    char *str = (char *) ________;        // (1) cast the arg pointer
    printf("Thread received: %s\n", str);
    return (void *) (long) ________;      // (2) return length as void*
}

int main() {
    pthread_t tid;
    char *msg = "pthread";
    void *retval;

    // (3) Create the thread — fill in the function pointer and argument
    int result = pthread_create(&tid, NULL, ________, ________);
    if (result != 0) {
        perror("pthread_create");
        return 1;
    }

    // (4) Wait for thread to finish and get its return value
    pthread_join(________, &retval);

    printf("Thread returned: %ld\n", (long) retval);
    return 0;
}
```

Fill in blanks (1)–(4). What compiler flag is required to compile this program?

```
Compiler flag: _________________________
```

<details><summary>▶ Answer</summary>

```c
// (1)
char *str = (char *) arg;

// (2)
return (void *) (long) strlen(str);

// (3)
int result = pthread_create(&tid, NULL, thread_func, (void *) msg);

// (4)
pthread_join(tid, &retval);
```

**Compiler flag:** `-pthread` (e.g., `clang -pthread lab.c`)

This flag links the pthread library and enables thread-safety features. Without it, `pthread_create` will be undefined.

**Expected output:**
```
Thread received: pthread
Thread returned: 7
```

</details>

---

**Q18.** (4 pts) Consider **Round Robin scheduling** with a quantum of **3 ms**. Four processes arrive at the times shown:

| Process | Burst Time | Arrival Time |
|---------|-----------|--------------|
| P1      | 6 ms      | 0 ms         |
| P2      | 4 ms      | 1 ms         |
| P3      | 2 ms      | 2 ms         |
| P4      | 3 ms      | 3 ms         |

**(a) (2 pts)** Draw the execution timeline (Gantt chart). Use the ready queue approach: P1 starts at t=0, add arriving processes to the queue when their arrival time comes.

```
Gantt:  [    ][ ][ ][ ][ ][ ]
Time:  0   3   6  8  10  12  13
```

**(b) (2 pts)** Calculate the **average turnaround time** for all four processes.

```
P1 turnaround: _____
P2 turnaround: _____
P3 turnaround: _____
P4 turnaround: _____
Average: _____
```

<details><summary>▶ Answer</summary>

**Execution trace (quantum = 3, FIFO within same time):**

Ready queue evolution:
- t=0: Queue: [P1]. Run P1 for 3ms
- t=1: P2 arrives → queue: [P2] (P1 still running)
- t=2: P3 arrives → queue: [P2, P3]
- t=3: P1 done quantum (3ms used, 3ms remaining). P4 arrives → queue: [P2, P3, P4, P1]
- Run P2 for 3ms (t=3→6). P2 remaining = 1ms
- t=6: P2 done quantum. Queue: [P3, P4, P1, P2]
- Run P3 for 2ms (t=6→8). P3 finishes!
- t=8: Queue: [P4, P1, P2]
- Run P4 for 3ms (t=8→11). P4 finishes!
- t=11: Queue: [P1, P2]
- Run P1 for 3ms (t=11→14). P1 finishes!
- t=14: Queue: [P2]
- Run P2 for 1ms (t=14→15). P2 finishes!

**Gantt:**
```
[P1: 0-3][P2: 3-6][P3: 6-8][P4: 8-11][P1: 11-14][P2: 14-15]
```

**Turnaround times (finish - arrival):**
- P1: 14 - 0 = **14 ms**
- P2: 15 - 1 = **14 ms**
- P3: 8 - 2 = **6 ms**
- P4: 11 - 3 = **8 ms**
- **Average = (14 + 14 + 6 + 8) / 4 = 42 / 4 = 10.5 ms**

</details>

---

**Q19.** (4 pts) The code below attempts to write a signal handler for `SIGINT` that safely displays a message without using `printf`. However, it has **three errors**. Identify each error and write the corrected version.

```c
#define _POSIX_C_SOURCE 200809L
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

// Error A: in this function
void handle_sigint(int signum) {
    printf("Ctrl-C caught\n");    // Line A
}

int main(void) {
    struct sigaction sa;
    sa.sa_flags = 0;
    // Error B: missing initialization
    sigaction(SIGINT, &sa, NULL); // Line B
    // Error C: missing handler assignment
    while (1) {
        sleep(1);
    }
}
```

**Error A (in handler):**

```


```

**Error B (before sigaction call):**

```


```

**Error C (before sigaction call):**

```


```

**Corrected complete program:**

```c


```

<details><summary>▶ Answer</summary>

**Error A:** `printf()` is not async-signal-safe. Must use `write()`.

**Error B:** `sa.sa_mask` is not initialized. Without `sigemptyset(&sa.sa_mask)`, the mask contains garbage, risking blocking random signals during handler execution.

**Error C:** `sa.sa_handler` is never assigned the handler function pointer — the struct has an uninitialized function pointer, causing undefined behavior.

**Corrected program:**

```c
#define _POSIX_C_SOURCE 200809L
#include <signal.h>
#include <string.h>
#include <unistd.h>

void handle_sigint(int signum) {
    // FIX A: use write() — signal-safe
    const char *msg = "Ctrl-C caught\n";
    write(STDOUT_FILENO, msg, strlen(msg));
}

int main(void) {
    struct sigaction sa;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);        // FIX B: initialize mask
    sa.sa_handler = handle_sigint;   // FIX C: assign handler
    sigaction(SIGINT, &sa, NULL);
    while (1) {
        sleep(1);
    }
    return 0;
}
```

</details>

---

## Answer Summary

| Q | Pts | Topic |
|---|-----|-------|
| 1 | 2 | sigaction struct |
| 2 | 2 | fork() behavior |
| 3 | 2 | Round Robin quantum |
| 4 | 2 | errno |
| 5 | 2 | Data race / threads |
| 6 | 2 | WIFEXITED / waitpid |
| 7 | 2 | External fragmentation |
| 8 | 2 | getline() vs fgets() |
| 9 | 2 | exec() never returns |
| 10 | 2 | Coalescing |
| 11 | 2 | Turnaround vs waiting time |
| 12 | 2 | Page fault |
| 13 | 2 | Systems programming examples |
| 14 | 2 | FCFS convoy effect |
| 15 | 2 | Variable memory segments |
| 16 | 4 | Code analysis / output prediction |
| 17 | 4 | pthread_create fill-in |
| 18 | 4 | Round Robin scheduling calc |
| 19 | 4 | Signal handler 3-error fix |
| **Total** | **46** | |

---
*CMPT 201 Practice Midterm 3 — Spring 2026*
