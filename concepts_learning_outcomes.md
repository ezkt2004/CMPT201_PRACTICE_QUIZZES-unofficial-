# A10: MapReduce — Concepts & Learning Outcomes

> Flashcard-style reference. Each concept has expandable details and examples
> drawn directly from the README and our `mapreduce.c` implementation.

---

## Learning Outcome 1 — The Three-Stage MapReduce Algorithm

> **Understand the three-stage MapReduce algorithm: what each stage takes as input, its job, and what it produces as output.**

---

### Stage Overview

| Stage | Input | Job | Output |
|---|---|---|---|
| **Map** | One `(key, value)` pair at a time from the input list | Call user's `map()` once per pair; user emits intermediate pairs | Set of intermediate `(key, value)` pairs |
| **Group-By-Key** | All intermediate pairs from all map threads | Sort by key; group values with same key together | List of `(key, [v1, v2, ...])` pairs |
| **Reduce** | One `(key, [v1, v2, ...])` grouped pair at a time | Call user's `reduce()` once per grouped pair; user emits final pairs | Final `(key, value)` output pairs |

---

<details>
<summary><strong>Flashcard: What does the MAP stage receive and produce?</strong></summary>

**Receives (per call):** A single `(key, value)` input pair — e.g., `("1", "the quick brown fox")`

**Job:** The MapReduce framework calls the user's `map()` once per input pair. The `map()` can emit zero or more intermediate key-value pairs via `mr_emit_i()`.

**Produces:** A set of intermediate `(key, value)` pairs — e.g., `("the","1"), ("quick","1"), ("brown","1"), ("fox","1")`

**In code (`mapreduce.c`):**
```c
// mapper_fn calls map() once per KV in the thread's chunk
for (size_t i = 0; i < a->count; i++) {
    a->map(&a->start[i]);   // framework calls map; map calls mr_emit_i()
}
```

</details>

---

<details>
<summary><strong>Flashcard: What does the GROUP-BY-KEY stage receive and produce?</strong></summary>

**Receives:** All intermediate pairs from all map threads (stored in `g_inter[]`)

**Job:**
1. Sort all intermediate pairs by key alphabetically
2. Identify unique keys and group all values under each unique key

**Produces:** Grouped pairs of the form `(key, [val1, val2, ...])` — e.g., `("the", ["1","1","1","1"])`

**In code (`mapreduce.c`):**
```c
// Step 1: sort
qsort(g_inter, g_inter_n, sizeof(struct mr_in_kv), cmp_in_kv);

// Step 2: group (two-pass: count unique keys, then allocate and fill)
struct mr_out_kv *grouped = malloc(unique * sizeof(struct mr_out_kv));
// ... walk sorted array building grouped[]
```

No threads are involved — this is a serial step done by the framework between map and reduce.

</details>

---

<details>
<summary><strong>Flashcard: What does the REDUCE stage receive and produce?</strong></summary>

**Receives (per call):** A single grouped pair `(key, [v1, v2, ...])` — e.g., `("the", ["1","1","1","1"])`

**Job:** The MapReduce framework calls the user's `reduce()` once per grouped pair. The `reduce()` can emit zero or more final key-value pairs via `mr_emit_f()`.

**Produces:** Final `(key, value)` pairs — e.g., `("the", "4")`

**In code:**
```c
for (size_t i = 0; i < a->count; i++) {
    a->reduce(&a->start[i]);  // framework calls reduce; reduce calls mr_emit_f()
}
```

</details>

---

<details>
<summary><strong>Flashcard: Why is there a sort/group step between map and reduce?</strong></summary>

The `map` stage produces raw intermediate pairs — many different threads emit pairs with
overlapping keys in arbitrary order. Before `reduce()` can meaningfully aggregate values
for a key, all values for that key must be collected together.

Sorting groups identical keys contiguously → a single linear scan can build the grouped
`(key, [values])` structure without any hash table.

The README defines this as the **group-by-key** stage, and it is entirely the framework's
responsibility (not the user's).

</details>

---

## Learning Outcome 2 — Analyzing a MapReduce Example

> **Able to analyze an example use of the MapReduce algorithm.**

---

<details>
<summary><strong>Flashcard: Full word-count worked example (4 inputs, 2 map threads)</strong></summary>

**Input:**
```
("1", "the quick brown fox")
("2", "jumps over the lazy dog")
("3", "the quick brown fox")
("4", "jumps over the lazy dog")
```

**Map stage (user's map emits (word, "1") for each word):**

Thread 0 processes chunk 0 — inputs 1 & 2:
```
("the","1") ("quick","1") ("brown","1") ("fox","1")
("jumps","1") ("over","1") ("the","1") ("lazy","1") ("dog","1")
```
Thread 1 processes chunk 1 — inputs 3 & 4: (same output)

**Group-by-key stage (framework sorts & groups):**
```
("brown", ["1","1"])
("dog",   ["1","1"])
("fox",   ["1","1"])
("jumps", ["1","1"])
("lazy",  ["1","1"])
("over",  ["1","1"])
("quick", ["1","1"])
("the",   ["1","1","1","1"])
```

**Reduce stage (user's reduce counts elements in value list):**
```
("brown","2") ("dog","2") ("fox","2") ("jumps","2")
("lazy","2")  ("over","2") ("quick","2") ("the","4")
```

**Final output:** sorted alphabetically, matching the above.

</details>

---

<details>
<summary><strong>Flashcard: What does amr_map and amr_reduce do in map_and_reduce.c?</strong></summary>

```c
// The "key" of each input is a line number; the "value" is a word.
// map: flip — use the word as the new key, emit "1" as the value.
void amr_map(const struct mr_in_kv *in_kv) {
    mr_emit_i(in_kv->value, "1");
    //         ^-- word becomes the intermediate KEY
    //                      ^-- always "1" (record an encounter)
}

// reduce: count how many "1"s are associated with this word.
void amr_reduce(const struct mr_out_kv *inter_kv) {
    size_t cnt = 0;
    for (size_t i = 0; i < inter_kv->count; i++) {
        if (strcmp(inter_kv->value[i], "1") == 0) cnt++;
    }
    char cnt_str[MAX_VALUE_SIZE];
    snprintf(cnt_str, MAX_VALUE_SIZE, "%zu", cnt);
    mr_emit_f(inter_kv->key, cnt_str);  // emit (word, "count")
}
```

The key insight: **the map swaps key and value**. The input key (line number) is
discarded; the input value (word) becomes the intermediate key.

</details>

---

<details>
<summary><strong>Flashcard: What does "framework treats map/reduce as a black box" mean?</strong></summary>

The framework (`mr_exec`, `mr_emit_i`, `mr_emit_f`) does not look inside the user's
map or reduce functions. It just:
1. Calls `map(input_pair)` — the user decides what intermediates to emit
2. Collects intermediates, sorts, and groups them
3. Calls `reduce(grouped_pair)` — the user decides what finals to emit

The user can implement *any* logic in map and reduce. The framework is general-purpose.
Examples from the README: finding lines containing a word, counting URL access frequency,
listing all pages linking to a given URL — all use the same framework with different
map/reduce functions.

</details>

---

## Learning Outcome 3 — Passing Array Slices to Threads via Pointers

> **Understand how to work with pointers to pass different threads part of an array.**

---

<details>
<summary><strong>Flashcard: How do we give each thread its own slice without copying?</strong></summary>

We use **pointer arithmetic** to point into a contiguous array:

```c
// input->kv_lst is a flat array: [kv0, kv1, kv2, kv3, kv4, kv5, kv6]
//
// With 3 threads on 7 elements: thread 0 → [0,2], thread 1 → [3,4], thread 2 → [5,6]

size_t offset = 0;
for (size_t i = 0; i < mapper_count; i++) {
    args[i].start = &input->kv_lst[offset];  // pointer to element at [offset]
    args[i].count = base + (i < extra ? 1 : 0);
    offset += args[i].count;
}
```

`&input->kv_lst[offset]` is equivalent to `input->kv_lst + offset` — both give a
pointer to the element at index `offset` in the array. No data is copied; the thread
receives a pointer into the *same* memory.

Inside `mapper_fn`, the thread accesses its slice:
```c
for (size_t i = 0; i < a->count; i++) {
    a->map(&a->start[i]);   // a->start[i] is element at original_index[offset + i]
}
```

This is safe because **mapper threads only read** from the slice (they don't write to
input), so no synchronization is needed for the slice access itself.

</details>

---

<details>
<summary><strong>Flashcard: Why must thread argument structs be heap-allocated?</strong></summary>

```c
// WRONG — stack allocated:
struct mapper_arg args[mapper_count];  // variable-length array on the stack

// After pthread_create, if the creating function returns before threads finish,
// the stack frame is gone → the thread reads garbage memory.
```

```c
// CORRECT — heap allocated:
struct mapper_arg *args = malloc(mapper_count * sizeof(struct mapper_arg));

// Heap memory persists until explicitly freed.
// We free AFTER pthread_join (after all threads have finished).
free(args);  // safe — threads are all done
```

VLAs (variable-length arrays) on the stack would be dangerous for thread arguments
because the thread might still be running after the creating function's stack frame
is gone. Heap allocation (`malloc`) is the safe choice.

</details>

---

<details>
<summary><strong>Flashcard: What is the memory layout when we pass a slice pointer?</strong></summary>

```
input->kv_lst:
 Index: [  0  ][  1  ][  2  ][  3  ][  4  ][  5  ][  6  ]  (7 elements)
              ^                ^               ^
              |                |               |
        args[0].start   args[1].start   args[2].start
        count=3          count=2          count=2

Thread 0 sees: kv_lst[0], kv_lst[1], kv_lst[2]
Thread 1 sees: kv_lst[3], kv_lst[4]
Thread 2 sees: kv_lst[5], kv_lst[6]
```

All three pointers point into the *same* original array — no copies made.
Each `struct mapper_arg` stores a pointer + a length, like a slice descriptor.

</details>

---

## Learning Outcome 4 — POSIX Threads: Create and Join

> **Able to use POSIX threads: create, join.**

---

<details>
<summary><strong>Flashcard: pthread_create — full signature and usage</strong></summary>

```c
#include <pthread.h>

int pthread_create(
    pthread_t *thread,                  // [OUT] stores the new thread's ID
    const pthread_attr_t *attr,         // thread attributes; NULL = defaults
    void *(*start_routine)(void *),     // [IN] function the thread runs
    void *arg                           // [IN] argument passed to start_routine
);
// Returns 0 on success, non-zero error code on failure
```

**Our usage:**
```c
pthread_t tids[mapper_count];

for (size_t i = 0; i < mapper_count; i++) {
    pthread_create(&tids[i], NULL, mapper_fn, &args[i]);
    //              ^         ^     ^           ^
    //              |         |     |           argument to mapper_fn
    //              |         |     entry function
    //              |         default attrs
    //              output: thread ID stored here
}
```

After all `pthread_create` calls, all mapper threads are running **concurrently**.

</details>

---

<details>
<summary><strong>Flashcard: pthread_join — full signature and why we need it</strong></summary>

```c
int pthread_join(
    pthread_t thread,   // the thread to wait for
    void **retval       // [OUT] captures thread's return value; NULL if unused
);
// Blocks calling thread until `thread` terminates
// Returns 0 on success
```

**Our usage:**
```c
for (size_t i = 0; i < mapper_count; i++) {
    pthread_join(tids[i], NULL);  // wait for thread i to finish
}
// After this loop: ALL map threads have completed
// SAFE to access g_inter[] now
```

**Why join before group-by-key?**
If we proceeded to sort `g_inter[]` while map threads were still running,
they could still be writing to it → data race → undefined behavior.
`pthread_join` is our synchronization barrier.

</details>

---

<details>
<summary><strong>Flashcard: Thread entry function requirements</strong></summary>

POSIX requires thread entry functions to have this exact signature:

```c
void *my_thread_fn(void *arg);
//    ^               ^
//    returns void*   takes void* (type-erased argument)
```

Inside, we immediately cast `void *` back to our concrete type:

```c
void *mapper_fn(void *arg) {
    struct mapper_arg *a = (struct mapper_arg *)arg;  // cast void* → our type
    for (size_t i = 0; i < a->count; i++) {
        a->map(&a->start[i]);
    }
    return NULL;  // return value captured by pthread_join's second arg (we use NULL)
}
```

You **cannot** have a thread entry function with a different signature — the compiler
may accept it but pthread_create expects `void *(*)(void *)`.

</details>

---

## Learning Outcome 5 — POSIX Mutexes

> **Able to use POSIX mutexes.**

---

<details>
<summary><strong>Flashcard: The three mutex operations and when to use them</strong></summary>

```c
// DECLARE & INITIALIZE (static — at global scope)
pthread_mutex_t my_mutex = PTHREAD_MUTEX_INITIALIZER;

// OR dynamic initialization (for non-static):
pthread_mutex_t my_mutex;
pthread_mutex_init(&my_mutex, NULL);  // must call pthread_mutex_destroy() later

// LOCK — blocks until the mutex is available, then acquires it
pthread_mutex_lock(&my_mutex);

// CRITICAL SECTION — only one thread at a time executes this code
// ... shared data access ...

// UNLOCK — releases the mutex so another thread can acquire it
pthread_mutex_unlock(&my_mutex);
```

**Our usage in mr_emit_i:**
```c
static pthread_mutex_t g_inter_mu = PTHREAD_MUTEX_INITIALIZER;

int mr_emit_i(const char *key, const char *value) {
    pthread_mutex_lock(&g_inter_mu);    // acquire lock
    // --- CRITICAL SECTION ---
    if (g_inter_n == g_inter_cap) {
        g_inter_cap = g_inter_cap ? g_inter_cap * 2 : 64;
        g_inter = realloc(g_inter, g_inter_cap * sizeof(struct mr_in_kv));
    }
    strncpy(g_inter[g_inter_n].key,   key,   MAX_KEY_SIZE);
    strncpy(g_inter[g_inter_n].value, value, MAX_VALUE_SIZE);
    g_inter_n++;
    // --- END CRITICAL SECTION ---
    pthread_mutex_unlock(&g_inter_mu);  // release lock
    return 0;
}
```

</details>

---

<details>
<summary><strong>Flashcard: What data race does the mutex prevent here?</strong></summary>

Without the mutex, two map threads could interleave like this:

```
Thread A reads: g_inter_n == g_inter_cap → needs realloc
Thread B reads: g_inter_n == g_inter_cap → needs realloc (same!)
Thread A calls realloc → old pointer freed, new block returned
Thread B calls realloc with OLD (dangling) pointer → undefined behavior!
```

Or even simpler:
```
Thread A reads g_inter_n = 5, prepares to write to g_inter[5]
Thread B reads g_inter_n = 5, prepares to write to g_inter[5]
Both write to index 5 → one write clobbers the other
Both increment g_inter_n → g_inter_n = 7 but we only wrote to index 5 and 6
(one slot was lost, one was overwritten)
```

The mutex prevents all of this by making the entire check+realloc+write+increment
sequence atomic from the perspective of other threads.

</details>

---

<details>
<summary><strong>Flashcard: PTHREAD_MUTEX_INITIALIZER vs pthread_mutex_init()</strong></summary>

| | `PTHREAD_MUTEX_INITIALIZER` | `pthread_mutex_init()` |
|---|---|---|
| When to use | Static or global `pthread_mutex_t` variables | Dynamic (heap-allocated) or local mutex variables |
| Initialization | At declaration time, no function call needed | Function call required |
| Cleanup | Not strictly required (OS reclaims on exit) | Must call `pthread_mutex_destroy()` |
| Example | `static pthread_mutex_t mu = PTHREAD_MUTEX_INITIALIZER;` | `pthread_mutex_init(&mu, NULL);` |

In our code, both mutexes are static globals → `PTHREAD_MUTEX_INITIALIZER` is correct.

</details>

---

<details>
<summary><strong>Flashcard: Why is there no mutex in mapper_fn or reducer_fn for reading the slice?</strong></summary>

Mutexes protect **shared mutable state** — data that multiple threads read AND write.

The input array (`input->kv_lst`) and grouped array are **read-only** from the threads'
perspective. No thread modifies them. Multiple readers can access the same memory
simultaneously without a data race. Only writes to shared state (`g_inter[]`,`g_final[]`)
require mutual exclusion.

Rule of thumb:
- **Multiple threads writing** → mutex required
- **Multiple threads reading, nobody writing** → no mutex needed
- **One thread writing, others reading** → mutex (or read-write lock)

</details>

---

## Learning Outcome 6 — Function Pointers

> **Able to use function pointers: write code for declaring a variable, passing to a function, calling.**

---

<details>
<summary><strong>Flashcard: How to declare a function pointer variable</strong></summary>

**Syntax:** `return_type (*variable_name)(parameter_types);`

```c
// Pointer to a function that takes const struct mr_in_kv * and returns void
void (*map_ptr)(const struct mr_in_kv *);

// Pointer to a function that takes const struct mr_out_kv * and returns void
void (*reduce_ptr)(const struct mr_out_kv *);

// Pointer to a function that takes (const void*, const void*) and returns int
int (*cmp_ptr)(const void *, const void *);
```

**Reading the declaration:**
`void (*map_ptr)(const struct mr_in_kv *)`:
1. `map_ptr` is a variable
2. `(*map_ptr)` — it's a pointer
3. `(*map_ptr)(const struct mr_in_kv *)` — it's a pointer to a function taking that arg
4. `void (...)` — the function returns void

</details>

---

<details>
<summary><strong>Flashcard: How to pass a function pointer to a function</strong></summary>

**In mr_exec's signature:**
```c
int mr_exec(
    const struct mr_input *input,
    void (*map)(const struct mr_in_kv *),    // function pointer parameter
    size_t mapper_count,
    void (*reduce)(const struct mr_out_kv *), // function pointer parameter
    size_t reducer_count,
    struct mr_output *output
);
```

**Calling mr_exec (from test code):**
```c
void amr_map(const struct mr_in_kv *in_kv) { ... }
void amr_reduce(const struct mr_out_kv *inter_kv) { ... }

// Pass function names — they decay to function pointers automatically
mr_exec(&input, amr_map, 1, amr_reduce, 1, &output);
//              ^         function name acts as a pointer
```

No `&` required before function name (though `&amr_map` is also valid).

</details>

---

<details>
<summary><strong>Flashcard: How to call a function through a pointer</strong></summary>

```c
// Given: void (*map)(const struct mr_in_kv *);
// Assigned: args[i].map = map;  (stored in struct)

// Method 1: Direct call (most common, cleaner)
args[i].map(&args[i].start[j]);

// Method 2: Explicit dereference (equivalent, more verbose)
(*args[i].map)(&args[i].start[j]);
```

Both forms are valid C. Method 1 reads like a regular function call.

**As used in our code:**
```c
void *mapper_fn(void *arg) {
    struct mapper_arg *a = (struct mapper_arg *)arg;
    for (size_t i = 0; i < a->count; i++) {
        a->map(&a->start[i]);   // calling through function pointer stored in struct
    }
    return NULL;
}
```

</details>

---

<details>
<summary><strong>Flashcard: Why use function pointers instead of just calling the function directly?</strong></summary>

Function pointers enable **generality / polymorphism** in C:

Without function pointers, we would need a separate `mr_exec_word_count()`,
`mr_exec_url_finder()`, etc. — one for each use case.

With function pointers:
```c
// Same framework, different behaviors just by passing different functions
mr_exec(&input, word_count_map,  1, word_count_reduce,  1, &output);
mr_exec(&input, url_finder_map,  2, url_finder_reduce,  4, &output);
mr_exec(&input, log_analyzer_map,4, log_analyzer_reduce,8, &output);
```

The framework doesn't know or care *what* these functions do internally — it just calls
them at the right time with the right arguments. This is the **black box** principle
described in the README.

For `pthread_create`, the same principle applies: the thread entry function is passed
as a function pointer (`void *(*start_routine)(void *)`) allowing any function
(mapper_fn or reducer_fn) to be the thread's entry point.

</details>

---

<details>
<summary><strong>Flashcard: typedef for function pointers — cleaner syntax (concept)</strong></summary>

You can use `typedef` to give a function pointer type a name (not used in our code but
good to know):

```c
// Without typedef:
void (*map)(const struct mr_in_kv *);

// With typedef:
typedef void (*map_fn_t)(const struct mr_in_kv *);
map_fn_t map;   // much cleaner

// In struct:
struct mapper_arg {
    const struct mr_in_kv *start;
    size_t count;
    map_fn_t map;   // cleaner than: void (*map)(const struct mr_in_kv *);
};
```

In our actual code we used the full declaration form (without typedef) as seen in
`struct mapper_arg { ... void (*map)(const struct mr_in_kv *); ... }`.

</details>

---

## Quick Reference: Important Constants and Sizes

<details>
<summary><strong>Flashcard: Key sizes and struct layout on a 64-bit system</strong></summary>

```
MAX_KEY_SIZE   = 16 bytes
MAX_VALUE_SIZE = 16 bytes

struct mr_in_kv:
  char key[16]    → 16 bytes
  char value[16]  → 16 bytes
  ─────────────────────────
  Total           = 32 bytes  (no padding needed — both members natural aligned)

struct mr_out_kv:
  char key[16]              → 16 bytes
  char (*value)[16]         →  8 bytes (pointer on 64-bit)
  size_t count              →  8 bytes (size_t = 8 bytes on 64-bit)
  ────────────────────────────────────
  Total                     = 32 bytes  (16 + 8 + 8 — naturally aligned, no padding)

struct mr_input:
  struct mr_in_kv *kv_lst   →  8 bytes (pointer)
  size_t count              →  8 bytes
  ─────────────────────────
  Total                     = 16 bytes

struct mr_output:
  struct mr_out_kv *kv_lst  →  8 bytes (pointer)
  size_t count              →  8 bytes
  ─────────────────────────
  Total                     = 16 bytes
```

Note: `sizeof(char[MAX_VALUE_SIZE])` = `sizeof(char[16])` = 16.
So `malloc(run * sizeof(char[MAX_VALUE_SIZE]))` allocates `run * 16` bytes,
giving an array of `run` strings each of length 16.

</details>

---

<details>
<summary><strong>Flashcard: Tests.h constants</strong></summary>

```c
#define MAX_DATA_SIZE  1024   // max number of input KV pairs in tests
#define MAX_THREADS    32     // max number of threads tested
```

The `full_map_reduce()` test uses mappers and reducers as powers of 2:
`2, 4, 8, 16, 32` — iterating with `1 << (i+1)` for i in 0..4.

</details>

---

## Concurrency Model Summary

<details>
<summary><strong>Flashcard: What runs concurrently and what runs serially?</strong></summary>

```
mr_exec() ─────────────────────────────────────────────────────►
                │
                ▼ SERIAL: partition input, malloc args, malloc tids
                │
              ┌─┴──────────────────────┐
 CONCURRENT:  │ mapper thread 0        │ ... mapper thread N │
              │  calls map() per KV    │                     │
              └─────────────────────────────────────────────┘
                │
                ▼ SERIAL: pthread_join (wait for all mappers)
                │
                ▼ SERIAL: qsort g_inter
                │
                ▼ SERIAL: count unique keys, build grouped[]
                │
              ┌─┴──────────────────────┐
 CONCURRENT:  │ reducer thread 0       │ ... reducer thread M │
              │  calls reduce() per    │                      │
              │  grouped kv            │                      │
              └──────────────────────────────────────────────┘
                │
                ▼ SERIAL: pthread_join (wait for all reducers)
                │
                ▼ SERIAL: qsort g_final, build output->kv_lst
                │
              return 0
```

The group-by-key stage **cannot** be parallelized in our implementation because it
requires seeing ALL intermediates from ALL map threads. This is why we join all mappers
before sorting.

</details>
