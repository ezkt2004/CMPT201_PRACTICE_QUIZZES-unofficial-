# A10: MapReduce — Full Code Implementation Recap

> A detailed, annotated walkthrough of every file we wrote and why we wrote it the way we did.
> Think of this as a commented tour guide of the entire codebase.

---

## Table of Contents
1. [Project Layout](#1-project-layout)
2. [interface.h — The Contracts We Implement](#2-interfaceh--the-contracts-we-implement)
3. [mapreduce.c — Our Core Implementation](#3-mapreducec--our-core-implementation)
   - [Global State](#31-global-state)
   - [Thread Argument Structs](#32-thread-argument-structs)
   - [Thread Entry Functions](#33-thread-entry-functions)
   - [The Comparator](#34-the-comparator)
   - [mr_exec — The Big Orchestrator](#35-mr_exec--the-big-orchestrator)
   - [mr_emit_i and mr_emit_f](#36-mr_emit_i-and-mr_emit_f)
4. [Test Files — How They Probe Our Framework](#4-test-files--how-they-probe-our-framework)
5. [free_output.c — Memory Cleanup](#5-free_outputc--memory-cleanup)
6. [Key Implementation Decisions](#6-key-implementation-decisions)
7. [Data Flow Summary](#7-data-flow-summary)

---

## 1. Project Layout

```
a10-ezkt2004-main/
├── include/
│   ├── interface.h      ← defines structs + the 3 functions we must implement
│   └── tests.h          ← test macros and test function declarations
└── src/
    ├── mapreduce.c       ← OUR implementation (the file we wrote)
    ├── main.c            ← test runner provided by assignment
    ├── free_output.c     ← helper to free output memory (provided)
    ├── single_map.c      ← test: map called correctly with 1 thread
    ├── single_reduce.c   ← test: reduce called correctly with 1 thread
    ├── map_and_reduce.c  ← test: full word-count integration test
    ├── number_of_mappers_reducers.c ← test: correct thread counts
    └── partition.c       ← test: input and intermediate chunking
```

The rule: **we only wrote `mapreduce.c`**. Everything else was provided. The grader replaces
the provided files with its own versions, so we must not touch them.

---

## 2. interface.h — The Contracts We Implement

```c
#define MAX_KEY_SIZE   16   // every key is at most 16 chars (including \0)
#define MAX_VALUE_SIZE 16   // every value is at most 16 chars (including \0)
```

These constants are **baked into struct sizes**. They are NOT variable — the structs use
fixed-size character arrays, not pointers. This is critical for understanding memory layout.

---

### Struct: `mr_in_kv` — A flat input key-value pair

```c
struct mr_in_kv {
    char key[MAX_KEY_SIZE];     // 16 bytes
    char value[MAX_VALUE_SIZE]; // 16 bytes
};
// Total: 32 bytes, no padding needed (all char arrays, already byte-aligned)
```

This is used for:
- The raw **input** to the whole framework
- **Intermediate** results emitted by map threads (stored in our`g_inter[]` dynamic array)
- The **flat** (ungrouped) final results before we build the output structure

Why flat char arrays instead of `char *`? Because the assignment defined it this way.
Flat arrays make `malloc`/`realloc` simpler (one allocation covers the whole struct).

---

### Struct: `mr_out_kv` — A grouped key with multiple values

```c
struct mr_out_kv {
    char key[MAX_KEY_SIZE];        // 16 bytes — the unique key
    char (*value)[MAX_VALUE_SIZE]; // 8 bytes  — pointer to an array of strings
    size_t count;                  // 8 bytes  — how many values are in value[]
};
// Total: 32 bytes on 64-bit (16 + 8 + 8, naturally aligned)
```

`char (*value)[MAX_VALUE_SIZE]` is a **pointer to an array of 16 chars**.
- Dereferencing `value[i]` gives you the i-th string (a `char[16]`).
- The memory for the array is `malloc(count * sizeof(char[MAX_VALUE_SIZE]))`.

This distinction matters: `char *value[N]` would be an array of pointers;
`char (*value)[N]` is a pointer to arrays. They are very different layouts.

---

### Struct: `mr_input` and `mr_output`

```c
struct mr_input {
    struct mr_in_kv *kv_lst;  // pointer to array of input kv pairs
    size_t count;             // how many
};

struct mr_output {
    struct mr_out_kv *kv_lst; // pointer to array of output kv pairs (grouped)
    size_t count;             // how many unique keys
};
```

The caller owns `mr_input`'s memory. **We** are responsible for populating `mr_output`
inside `mr_exec` (we `malloc` it) and the caller frees it via `free_output()`.

---

### Functions we implement

```c
int mr_exec(const struct mr_input *input,
            void (*map)(const struct mr_in_kv *),
            size_t mapper_count,
            void (*reduce)(const struct mr_out_kv *),
            size_t reducer_count,
            struct mr_output *output);

int mr_emit_i(const char *key, const char *value);
int mr_emit_f(const char *key, const char *value);
```

Note: `map` and `reduce` are **function pointers** passed in, not our functions.
We just call them — we don't define their logic. The developer (test code) defines them.

---

## 3. mapreduce.c — Our Core Implementation

### 3.1 Global State

```c
// --- INTERMEDIATE storage (populated by map threads via mr_emit_i) ---
static struct mr_in_kv *g_inter    = NULL;   // dynamic array of flat kv pairs
static size_t           g_inter_n  = 0;      // current count
static size_t           g_inter_cap = 0;     // current capacity
static pthread_mutex_t  g_inter_mu = PTHREAD_MUTEX_INITIALIZER;

// --- FINAL output storage (populated by reduce threads via mr_emit_f) ---
static struct mr_in_kv *g_final    = NULL;
static size_t           g_final_n  = 0;
static size_t           g_final_cap = 0;
static pthread_mutex_t  g_final_mu = PTHREAD_MUTEX_INITIALIZER;
```

**Why globals?**
`mr_emit_i` and `mr_emit_f` are called from *inside the user's map/reduce functions*.
The user's functions have no access to our internal state — they can only call the
`mr_emit_*` API. So these arrays must be globally accessible from the emit functions.

**Why `static`?**
The `static` keyword at file scope means these globals are **only visible within
`mapreduce.c`**. This is good encapsulation — no other translation unit can accidentally
touch them.

**Why `PTHREAD_MUTEX_INITIALIZER`?**
This is the **static initializer** for a mutex. It works for statically-allocated
`pthread_mutex_t` variables (i.e., global or static locals). It is equivalent to calling
`pthread_mutex_init(&mu, NULL)` but requires no cleanup call. Since multiple map threads
will call `mr_emit_i` concurrently, the mutex prevents data races on `g_inter`.

**Why dynamic arrays (realloc doubling)?**
We don't know up front how many intermediate pairs map will produce. Doubling amortizes
the cost of reallocation to O(1) per insertion on average.

---

### 3.2 Thread Argument Structs

```c
// Passed to each mapper thread
struct mapper_arg {
    const struct mr_in_kv *start;        // pointer to first KV in this thread's chunk
    size_t count;                        // how many KVs in this thread's chunk
    void (*map)(const struct mr_in_kv *); // function pointer to the user's map()
};

// Passed to each reducer thread
struct reducer_arg {
    const struct mr_out_kv *start;           // pointer to first grouped KV in chunk
    size_t count;                            // how many grouped KVs in this chunk
    void (*reduce)(const struct mr_out_kv *); // function pointer to user's reduce()
};
```

**Why pack the function pointer into the struct?**
`pthread_create` gives us only a single `void *` argument. By packing everything the
thread needs into a struct and passing a pointer to it, we can convey multiple parameters.

**Why `start` as a pointer, not a copy?**
We are pointing directly into the *existing* arrays (`input->kv_lst` and `grouped[]`).
This is zero-copy — threads read their slice of shared memory. Since threads only read
(no writes to the input data), there is no data race on the slices themselves.

Pointer arithmetic to get slices:
```c
args[i].start = &input->kv_lst[offset];
// offset starts at 0 and accumulates; each thread gets a contiguous sub-array
```

---

### 3.3 Thread Entry Functions

```c
// mapper_fn is the function run by each mapper pthread
void *mapper_fn(void *arg) {
    struct mapper_arg *a = (struct mapper_arg *)arg;  // cast void* back to our type
    for (size_t i = 0; i < a->count; i++) {
        a->map(&a->start[i]);  // call user's map() for each kv in this chunk
    }
    return NULL;               // pthreads requires void* return; NULL means no retval
}

void *reducer_fn(void *arg) {
    struct reducer_arg *a = (struct reducer_arg *)arg;
    for (size_t i = 0; i < a->count; i++) {
        a->reduce(&a->start[i]);  // call user's reduce() for each grouped kv
    }
    return NULL;
}
```

**Thread entry functions must have signature `void *(fn)(void *)`** — this is the POSIX
requirement. We cast the `void *arg` back to our concrete struct type immediately.

**Calling the function pointer:**
```c
a->map(&a->start[i]);
// a->map  is a pointer to the function
// Calling it: just treat it like a regular function call
// Equivalent: (*a->map)(&a->start[i]);  — explicit dereference form
```

---

### 3.4 The Comparator

```c
static int cmp_in_kv(const void *a, const void *b) {
    return strncmp(((struct mr_in_kv *)a)->key,
                   ((struct mr_in_kv *)b)->key,
                   MAX_KEY_SIZE);
}
```

Required by `qsort`. The signature must be `int (const void *, const void *)`.
We cast the opaque pointers back to our type and compare keys using `strncmp`.

**Why `strncmp` with `MAX_KEY_SIZE` instead of `strcmp`?**
Defensive safety: protects against keys that might not be null-terminated within 16 bytes.
`strncmp` will never read past `MAX_KEY_SIZE` bytes.

This same comparator is reused for both `g_inter[]` (intermediate) and `g_final[]`
(final flat store) sorting, even though the final is semantically different. It works
because both arrays are `struct mr_in_kv[]` with the same layout.

---

### 3.5 mr_exec — The Big Orchestrator

Here is the full pipeline annotated step by step:

#### Step 0: Reset all state

```c
free(g_inter);      // free any leftover from a previous call
g_inter = NULL;
g_inter_n = 0;
g_inter_cap = 0;

free(g_final);
g_final = NULL;
g_final_n = 0;
g_final_cap = 0;

output->kv_lst = NULL;   // initialize output to empty
output->count  = 0;
```

This is essential for **supporting multiple calls** to `mr_exec`. Since globals persist
between calls, we must reset them. Without this, intermediate results from a previous
call would pollute the new run.

---

#### Step 1: Partition input and launch mapper threads

```c
// Calculate chunk sizes
size_t base  = input->count / mapper_count;  // floor division
size_t extra = input->count % mapper_count;  // remainder
size_t offset = 0;

// Heap-allocate thread IDs and argument structs
pthread_t       *tids = malloc(mapper_count * sizeof(pthread_t));
struct mapper_arg *args = malloc(mapper_count * sizeof(struct mapper_arg));

for (size_t i = 0; i < mapper_count; i++) {
    args[i].start = &input->kv_lst[offset];        // pointer into shared array
    args[i].count = base + (i < extra ? 1 : 0);    // first `extra` threads get +1
    args[i].map   = map;                           // store function pointer
    offset       += args[i].count;
    pthread_create(&tids[i], NULL, mapper_fn, &args[i]);
}
```

**Partitioning formula explained:**
If we have N items and T threads:
- `base = N / T` (minimum items per thread)
- `extra = N % T` (how many threads get one extra item)
- Thread 0..extra-1 get `base + 1` items; threads extra..T-1 get `base` items

Example: 7 items, 3 threads → base=2, extra=1
- Thread 0: 3 items, Thread 1: 2 items, Thread 2: 2 items

`pthread_create` signature:
```c
int pthread_create(pthread_t *thread,           // output: thread ID
                   const pthread_attr_t *attr,  // NULL → default attributes
                   void *(*start_routine)(void *), // thread entry function
                   void *arg);                  // argument passed to entry fn
```
Returns 0 on success. We pass `&args[i]` which is the address of each mapper_arg.
Each thread gets its own distinct struct in the `args` array.

---

#### Step 2: Join mapper threads

```c
for (size_t i = 0; i < mapper_count; i++) {
    pthread_join(tids[i], NULL);
}
```

`pthread_join` **blocks** the calling thread until thread `tids[i]` finishes.
Joining all mappers ensures all intermediate results are in `g_inter[]` before
we proceed to the group-by-key stage. The second argument would capture the
thread's return value — we pass NULL since we don't need it.

---

#### Step 3: Sort intermediates

```c
qsort(g_inter, g_inter_n, sizeof(struct mr_in_kv), cmp_in_kv);
```

`qsort` sorts in-place. After this, all entries with the same key are contiguous.
This is the sort step that makes the group-by-key stage a simple linear scan.

---

#### Step 4: Count unique keys (first pass)

```c
size_t unique = 0;
for (size_t i = 0; i < g_inter_n;) {
    size_t j = i + 1;
    while (j < g_inter_n &&
           strncmp(g_inter[i].key, g_inter[j].key, MAX_KEY_SIZE) == 0) {
        j++;  // advance j while keys match
    }
    unique++;  // found one unique key spanning [i, j)
    i = j;    // jump to start of next group
}
```

This walk counts unique keys so we know the exact size for `malloc(unique * sizeof(struct mr_out_kv))`.
We do two passes (count first, then fill) to avoid reallocation. The `i = j` jump skips
entire runs of identical keys in one step.

---

#### Step 5: Build grouped array (second pass)

```c
struct mr_out_kv *grouped = malloc(unique * sizeof(struct mr_out_kv));
size_t gi = 0;

for (size_t i = 0; i < g_inter_n;) {
    size_t j = i + 1;
    while (j < g_inter_n &&
           strncmp(g_inter[i].key, g_inter[j].key, MAX_KEY_SIZE) == 0)
        j++;
    size_t run = j - i;   // number of values for this key

    strncpy(grouped[gi].key, g_inter[i].key, MAX_KEY_SIZE);
    grouped[gi].count = run;
    // Allocate value array: an array of `run` strings, each of size MAX_VALUE_SIZE
    grouped[gi].value = malloc(run * sizeof(char[MAX_VALUE_SIZE]));
    for (size_t k = 0; k < run; k++) {
        strncpy(grouped[gi].value[k], g_inter[i + k].value, MAX_VALUE_SIZE);
    }

    gi++;
    i = j;
}
```

`malloc(run * sizeof(char[MAX_VALUE_SIZE]))` allocates a contiguous block of
`run * 16` bytes. `grouped[gi].value[k]` is a 16-byte slot for each copy of
a value string.

---

#### Step 6: Partition grouped array and launch reducer threads

```c
pthread_t         *r_tids = malloc(reducer_count * sizeof(pthread_t));
struct reducer_arg *r_args = malloc(reducer_count * sizeof(struct reducer_arg));

size_t r_base   = unique / reducer_count;
size_t r_extra  = unique % reducer_count;
size_t r_offset = 0;

for (size_t i = 0; i < reducer_count; i++) {
    r_args[i].start  = &grouped[r_offset];
    r_args[i].count  = r_base + (i < r_extra ? 1 : 0);
    r_args[i].reduce = reduce;
    r_offset        += r_args[i].count;
    pthread_create(&r_tids[i], NULL, reducer_fn, &r_args[i]);
}
```

Identical partitioning logic as the mapper stage, but applied to the `grouped[]` array
(which has `unique` entries). Each reducer thread gets a contiguous slice.

---

#### Step 7: Join reducer threads, free intermediate memory

```c
for (size_t i = 0; i < reducer_count; i++) {
    pthread_join(r_tids[i], NULL);
}

free(r_tids);
free(r_args);

// Free each grouped entry's value array (separately heap-allocated)
for (size_t i = 0; i < unique; i++) {
    free(grouped[i].value);
}
free(grouped);  // then free the grouped array itself
```

After joining, `g_final[]` contains all final flat kv pairs emitted by reduce threads.
We free the grouped array because it has served its purpose.

---

#### Step 8: Sort and build final output (mirrors Step 3-5 for finalresults)

```c
qsort(g_final, g_final_n, sizeof(struct mr_in_kv), cmp_in_kv);

// count unique keys in g_final
size_t unique_final = 0;
for (size_t i = 0; i < g_final_n;) {
    size_t j = i + 1;
    while (j < g_final_n &&
           strncmp(g_final[i].key, g_final[j].key, MAX_KEY_SIZE) == 0) j++;
    unique_final++;
    i = j;
}

// build output->kv_lst
output->kv_lst = malloc(unique_final * sizeof(struct mr_out_kv));
output->count  = unique_final;

size_t fi = 0;
for (size_t i = 0; i < g_final_n;) {
    size_t j = i + 1;
    while (j < g_final_n &&
           strncmp(g_final[i].key, g_final[j].key, MAX_KEY_SIZE) == 0) j++;
    size_t run = j - i;

    strncpy(output->kv_lst[fi].key, g_final[i].key, MAX_KEY_SIZE);
    output->kv_lst[fi].count = run;
    output->kv_lst[fi].value = malloc(run * sizeof(char[MAX_VALUE_SIZE]));
    for (size_t k = 0; k < run; k++) {
        strncpy(output->kv_lst[fi].value[k], g_final[i + k].value, MAX_VALUE_SIZE);
    }
    fi++;
    i = j;
}

free(tids);     // clean up mapper thread/arg arrays
free(args);
return 0;
```

The output is handed to the caller who must call `free_output()` when done.
Note: we sort `g_final` using `cmp_in_kv` (same comparator), which is valid because
`g_final` is also a `struct mr_in_kv[]`.

---

### 3.6 mr_emit_i and mr_emit_f

```c
int mr_emit_i(const char *key, const char *value) {
    pthread_mutex_lock(&g_inter_mu);   // CRITICAL SECTION START

    // Grow array if at capacity (doubling strategy)
    if (g_inter_n == g_inter_cap) {
        g_inter_cap = g_inter_cap ? g_inter_cap * 2 : 64;  // start at 64, then double
        g_inter = realloc(g_inter, g_inter_cap * sizeof(struct mr_in_kv));
    }

    // Copy key and value into the next slot
    strncpy(g_inter[g_inter_n].key,   key,   MAX_KEY_SIZE);
    strncpy(g_inter[g_inter_n].value, value, MAX_VALUE_SIZE);
    g_inter_n++;

    pthread_mutex_unlock(&g_inter_mu); // CRITICAL SECTION END
    return 0;
}
```

**Why the mutex is here and not in mapper_fn?**
Multiple mapper threads call `mr_emit_i` concurrently. The realloc + write to `g_inter`
is a non-atomic multi-step operation. Without the mutex:
- Two threads could simultaneously see `g_inter_n == g_inter_cap` and both realloc
- Two threads could write to the same index `g_inter_n` before either increments it

The mutex ensures only one thread executes the critical section at a time.

**Doubling growth:**
```c
g_inter_cap = g_inter_cap ? g_inter_cap * 2 : 64;
```
- If cap is 0 (first allocation), start with 64
- If cap is non-zero, double it
- `realloc(NULL, size)` is equivalent to `malloc(size)` — handles the initial NULL case

`mr_emit_f` is identical but uses `g_final` and `g_final_mu`.

---

## 4. Test Files — How They Probe Our Framework

### single_map.c
- Creates 1024 input KVs with sequential keys/values
- Calls mr_exec with 1 mapper, 1 reducer
- The `sm_map` function records every KV it receives (in call order)
- Verifies all 1024 KVs were received in order
- **Tests:** map is called once per input KV, in order, using 1 thread

### single_reduce.c
- Creates input where keys are `i % 8` (so 8 unique keys, 128 values each)
- The `sr_map` emits all KVs unchanged as intermediate
- The `sr_reduce` records what it receives grouped
- **Tests:** reduce is called once per unique key, with the right grouped values

### map_and_reduce.c
- Full word-count test with 1024 word-frequency input pairs
- `amr_map`: emits `(value, "1")` for each input (turns values into keys for counting)
- `amr_reduce`: counts how many "1"s per key → emits `(key, count_string)`
- Expected output: 57 unique words with precise counts
- `full_map_reduce()` tests 2,4,8,16,32 mappers × 2,4,8,16,32 reducers = 25 combinations

### number_of_mappers_reducers.c
- Uses `pthread_self()` inside map/reduce callbacks to detect how many unique threads ran
- **Tests:** exactly the right number of threads are created and used

### partition.c
- Tracks which thread processed which KVs using `pthread_self()`
- Verifies each thread got a contiguous, correctly-sized slice of the input
- **Tests:** partitioning formula is correct for both input and intermediate KVs

---

## 5. free_output.c — Memory Cleanup

```c
void free_output(struct mr_output *output) {
    if (output == NULL || output->kv_lst == NULL) return;

    for (size_t i = 0; i < output->count; i++) {
        free(output->kv_lst[i].value);   // free each value array (heap-alloc'd in mr_exec)
        output->kv_lst[i].value = NULL;  // null out to be safe
    }
    free(output->kv_lst);                // free the outer array
    output->kv_lst = NULL;
}
```

This mirrors how we `malloc` in `mr_exec`: we first free each `.value` pointer, then
free the outer `kv_lst` array. Order matters — you must free the inner allocations
before freeing the outer array (otherwise you'd lose the pointers).

---

## 6. Key Implementation Decisions

| Decision | What We Chose | Why |
|---|---|---|
| Where to store intermediates | Global dynamic arrays | `mr_emit_i` has no other way to communicate with `mr_exec` |
| Concurrent access protection | `pthread_mutex_t` per array | Multiple mapper threads write to `g_inter` simultaneously |
| Mutex initialization | `PTHREAD_MUTEX_INITIALIZER` | Static global — no need for `pthread_mutex_init` |
| Array growth | Doubling (×2), initial 64 | Amortized O(1) per push; 64 avoids tiny initial allocs |
| Sort before grouping | `qsort` by key | Makes grouping a single O(n) linear scan |
| Two-pass grouping | Count uniques, then fill | Know exact malloc size, no reallocation needed |
| Partition formula | `base + (i < extra ? 1 : 0)` | Distributes remainder evenly among first `extra` threads |
| Support multiple mr_exec calls | Reset globals at start | Globals persist; must clear old state |
| Thread args allocation | Heap (`malloc`) | Stack arrays would go out of scope before threads finish |

---

## 7. Data Flow Summary

```
INPUT (struct mr_input)
  │
  ▼
[PARTITION] ──────────────────────────────────────┐
  │  base = count/mappers, extra = count%mappers   │
  │  Thread i gets base + (i < extra ? 1 : 0) KVs │
  ▼                                                │
[MAPPER THREADS (x mapper_count)]                  │
  │  Each calls user's map() for each KV in chunk  │
  │  user's map() calls mr_emit_i(key, value)       │
  ▼                                                │
[g_inter[] — global dynamic array]                 │
  │  Protected by mutex                             │
  ▼                                                │
[SORT g_inter by key via qsort]                    │
  ▼                                                │
[GROUP BY KEY — two-pass linear scan]              │
  │  Produces: struct mr_out_kv grouped[]           │
  ▼                                                │
[PARTITION grouped]                                │
  ▼                                                │
[REDUCER THREADS (x reducer_count)]                │
  │  Each calls user's reduce() for each group      │
  │  user's reduce() calls mr_emit_f(key, value)    │
  ▼                                                │
[g_final[] — global dynamic array]                 │
  │  Protected by mutex                             │
  ▼                                                │
[SORT g_final by key via qsort]                    │
  ▼                                                │
[GROUP BY KEY — build output->kv_lst]              │
  ▼                                                │
OUTPUT (struct mr_output) ◄─────────────────────────┘
```
