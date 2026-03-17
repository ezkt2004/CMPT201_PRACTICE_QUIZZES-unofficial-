# A10: MapReduce — MCQ Practice Quiz

> Test yourself on the implementation, concepts, and technical details from A10.
> Each question has 4 options. Click the toggle below each question to reveal the
> answer and explanation.
>
> **Scoring:** 20 questions — 1 pt each. Set a timer: aim to finish in 25 minutes.

---

## Section 1 — Structs, Sizes, and Memory Layout

---

**Q1.** Given the definitions in `interface.h`:
```c
#define MAX_KEY_SIZE   16
#define MAX_VALUE_SIZE 16

struct mr_in_kv {
    char key[MAX_KEY_SIZE];
    char value[MAX_VALUE_SIZE];
};
```
What is `sizeof(struct mr_in_kv)` on a typical 64-bit system?

- A) 8 bytes
- B) 16 bytes
- C) 32 bytes
- D) 64 bytes

<details>
<summary>Reveal Answer</summary>

**Correct Answer: C — 32 bytes**

`char key[16]` = 16 bytes, `char value[16]` = 16 bytes. Both members are `char` arrays, the struct has no pointers or typed fields that require alignment padding between them. Total: 16 + 16 = **32 bytes**.

There is no padding here because no member requires alignment stricter than 1 byte (`char`). The struct's alignment requirement is 1, and its size is already a multiple of 1.

</details>

---

**Q2.** What does the field `char (*value)[MAX_VALUE_SIZE]` in `struct mr_out_kv` represent?

```c
struct mr_out_kv {
    char key[MAX_KEY_SIZE];
    char (*value)[MAX_VALUE_SIZE];
    size_t count;
};
```

- A) A pointer to a contiguous block of fixed-size strings (each of length `MAX_VALUE_SIZE`)
- B) An array of `MAX_VALUE_SIZE` char pointers
- C) A double pointer to char (equivalent to `char**`)
- D) A single string stored inline

<details>
<summary>Reveal Answer</summary>

**Correct Answer: A — A pointer to a contiguous block of fixed-size strings**

`char (*value)[MAX_VALUE_SIZE]` is a **pointer to an array of `MAX_VALUE_SIZE` chars**. Dereferencing with `value[i]` gives you the i-th string (a `char[16]`). The memory is allocated as:
```c
grouped[gi].value = malloc(run * sizeof(char[MAX_VALUE_SIZE]));
// = malloc(run * 16)  — a flat block of run×16 bytes
```

This is **not** the same as `char **value` (array of pointers). `char **value` would store pointers; `char (*value)[16]` stores the actual character data contiguously. This matters for memory layout and how subscripting works.

</details>

---

**Q3.** What is `sizeof(struct mr_out_kv)` on a 64-bit system where `sizeof(size_t) = 8` and `sizeof(void*) = 8`?

```c
struct mr_out_kv {
    char key[MAX_KEY_SIZE];        // ?
    char (*value)[MAX_VALUE_SIZE]; // ?
    size_t count;                  // ?
};
```

- A) 48 bytes
- B) 24 bytes
- C) 16 bytes
- D) 32 bytes

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — 32 bytes**

Layout on 64-bit:
- `char key[16]` = 16 bytes (alignment: 1 byte, no padding needed after it since next member has 8-byte alignment and 16 is divisible by 8)
- `char (*value)[16]` = 8 bytes (pointer, 8-byte aligned — no padding needed since offset 16 is already 8-byte aligned)
- `size_t count` = 8 bytes (8-byte aligned, offset 24 is 8-byte aligned)
- Total = 16 + 8 + 8 = **32 bytes** with no internal padding, and no trailing padding needed (struct size 32 is a multiple of max member alignment 8).

</details>

---

**Q4.** Why does `mr_emit_i` use `strncpy` instead of `strcpy` when copying keys and values?

- A) `strncpy` is faster due to SIMD optimizations
- B) `strncpy` limits the copy to at most `MAX_KEY_SIZE` bytes, preventing buffer overflows if the source string is longer than the fixed-size destination char array
- C) `strcpy` does not work with arrays that are fields in structs
- D) `strncpy` automatically null-terminates the destination in all cases

<details>
<summary>Reveal Answer</summary>

**Correct Answer: B — Buffer overflow prevention**

`strncpy(dest, src, n)` copies at most `n` bytes, ensuring we never write past the end of the `char[MAX_KEY_SIZE]` destination array regardless of how long `src` is. `strcpy` would blindly copy until it hits a null terminator in `src`, potentially overflowing the fixed-size array.

**Important caveat about D:** `strncpy` does **not** always null-terminate — if `src` is longer than `n`, the destination may not be null-terminated. Always be aware of this when using `strncpy`.

</details>

---

## Section 2 — Partitioning and Thread Dispatch

---

**Q5.** The framework partitions 7 input key-value pairs across 3 mapper threads. Using the formula `base = count / threads`, `extra = count % threads`, and thread `i` gets `base + (i < extra ? 1 : 0)` items — what sizes do threads 0, 1, and 2 receive?

- A) 2, 2, 3
- B) 2, 3, 2
- C) 1, 3, 3
- D) 3, 2, 2

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — 3, 2, 2**

`base = 7 / 3 = 2`, `extra = 7 % 3 = 1`

- Thread 0: `i=0 < extra=1` → `2 + 1 = 3`
- Thread 1: `i=1 < extra=1` is **false** → `2 + 0 = 2`
- Thread 2: `i=2 < extra=1` is **false** → `2 + 0 = 2`

Total: 3 + 2 + 2 = 7 ✓. The **first** `extra` threads get the extra item, ensuring maximum balance (no thread gets more than 1 extra item).

</details>

---

**Q6.** You have 5 unique intermediate keys (after the group-by-key stage) and 3 reducer threads. How are the grouped key-value pairs distributed?

- A) 2, 1, 2
- B) 1, 2, 2
- C) 2, 2, 2 — all equal (error case)
- D) 2, 2, 1

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — 2, 2, 1**

`base = 5 / 3 = 1`, `extra = 5 % 3 = 2`

- Thread 0: `0 < 2` → `1 + 1 = 2`
- Thread 1: `1 < 2` → `1 + 1 = 2`
- Thread 2: `2 < 2` is **false** → `1 + 0 = 1`

Total: 2 + 2 + 1 = 5 ✓. The last thread gets the smallest chunk.

</details>

---

**Q7.** Why are the `mapper_arg` structs heap-allocated (`malloc`) rather than declared as a VLA (variable-length array) on the stack inside `mr_exec`?

- A) `malloc` is faster than stack allocation for large structs
- B) VLAs cannot be used in functions that call `pthread_create`
- C) Stack memory inside `mr_exec` remains valid after `mr_exec` returns, while heap memory does not
- D) Each thread's argument struct must remain valid while the thread runs — heap memory persists until explicitly freed and won't be invalidated by any stack frame returning

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — Heap memory persists; stack frames can become invalid**

If `mapper_arg` structs were on `mr_exec`'s stack as a VLA, and if for any reason the frame were exited (or in a scenario where a thread outlives the function call), threads would read garbage from the destroyed stack frame. Heap allocating (`malloc`) ensures the struct data lives as long as we need it — we control lifetime by freeing after `pthread_join` (when we're certain all threads are done).

In our actual code `mr_exec` does not return early, but the heap approach is the correct and safe pattern.

</details>

---

**Q8.** In `partition.c`, the test tracks which thread processes which key-value pairs by using `pthread_self()` inside the map callback. Why is this call placed inside a `pthread_mutex_lock` / `pthread_mutex_unlock` block?

- A) `pthread_self()` is not thread-safe and requires a mutex to call safely
- B) `pthread_self()` is thread-safe, but the `partitions[]` array it writes to is shared — the mutex protects concurrent writes to that shared array from racing mapper threads
- C) The mutex is needed to call `pthread_equal()` correctly
- D) Without the mutex, `pthread_self()` would return the wrong thread ID

<details>
<summary>Reveal Answer</summary>

**Correct Answer: B — The shared partitions[] array needs protection**

`pthread_self()` **is** thread-safe (returns the calling thread's ID with no shared state). However, the `partitions[]` array in the test code is a global shared struct that multiple mapper threads write to simultaneously (each records its own thread ID and copied KVs). Without the mutex, two threads could write to the same partition slot simultaneously. The mutex protects the shared `partitions[]` from concurrent modification.

</details>

---

## Section 3 — POSIX Threads and Mutexes

---

**Q9.** What is the exact required signature for a thread entry function passed to `pthread_create`?

- A) `void thread_fn(void *arg)`
- B) `int thread_fn(void *arg)`
- C) `void *thread_fn(void *arg)`
- D) `void *thread_fn(pthread_t tid, void *arg)`

<details>
<summary>Reveal Answer</summary>

**Correct Answer: C — `void *thread_fn(void *arg)`**

POSIX specifies that the `start_routine` parameter of `pthread_create` must have the type `void *(*)(void *)` — a pointer to a function that takes `void *` and returns `void *`. Our implementations:
```c
void *mapper_fn(void *arg) { ... return NULL; }
void *reducer_fn(void *arg) { ... return NULL; }
```
We return `NULL` because we have no meaningful return value, but the return type must still be `void *`.

</details>

---

**Q10.** `pthread_join(tids[i], NULL)` is called after launching all mapper threads. What would happen if we skipped the join and immediately proceeded to `qsort(g_inter, ...)`?

- A) Nothing — `qsort` internally waits for threads to finish before sorting
- B) The sort would work correctly because threads finish before `qsort` reaches that data
- C) It would cause a compilation error
- D) It could produce a data race — mapper threads might still be writing to `g_inter[]` while `qsort` reads and reorders it, causing undefined behavior and incorrect results

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — Data race and undefined behavior**

`qsort` does not know about threads and does not wait for them. If any mapper thread is still calling `mr_emit_i()` (which writes to `g_inter[]`) while `qsort` is rearranging the same array, we have a **data race** — concurrent unsynchronized access to the same memory where at least one access is a write. The result is undefined behavior: corrupted data, crashes, or silently wrong output.

`pthread_join` is the explicit synchronization barrier guaranteeing all map threads have completed before we touch `g_inter[]`.

</details>

---

**Q11.** When should you use `PTHREAD_MUTEX_INITIALIZER` vs `pthread_mutex_init()`?

- A) `PTHREAD_MUTEX_INITIALIZER` is for statically-allocated (global or `static` local) `pthread_mutex_t` variables; `pthread_mutex_init()` is used for dynamically-allocated or locally-declared mutex variables
- B) `PTHREAD_MUTEX_INITIALIZER` is only available on Linux; `pthread_mutex_init()` is the portable POSIX version
- C) Both are identical — use whichever you prefer
- D) `PTHREAD_MUTEX_INITIALIZER` creates a recursive mutex; `pthread_mutex_init()` creates a non-recursive mutex

<details>
<summary>Reveal Answer</summary>

**Correct Answer: A — Static (global/static local) vs dynamic**

`PTHREAD_MUTEX_INITIALIZER` is a compile-time constant initializer that works only for **static storage duration** mutex variables (globals or `static` locals). For heap-allocated or automatic-storage (local non-static) mutexes, you must call `pthread_mutex_init()`.

In `mapreduce.c`:
```c
// Correct — static global:
static pthread_mutex_t g_inter_mu = PTHREAD_MUTEX_INITIALIZER;

// Would need pthread_mutex_init() for:
struct my_data { pthread_mutex_t mu; ... };
struct my_data *d = malloc(sizeof(*d));
pthread_mutex_init(&d->mu, NULL);  // must use this form
```

</details>

---

**Q12.** A programmer forgets to call `pthread_mutex_unlock(&g_inter_mu)` at the end of `mr_emit_i`. What is the consequence?

- A) The program compiles with a warning but runs correctly
- B) The first emit call succeeds, but all subsequent mapper threads that call `mr_emit_i` will block forever trying to acquire the already-held mutex — causing a deadlock
- C) The mutex automatically unlocks when the function returns
- D) Only the thread that held the lock is affected; others proceed normally

<details>
<summary>Reveal Answer</summary>

**Correct Answer: B — Deadlock**

After the first thread acquires `g_inter_mu` and returns without releasing it, the mutex is left in a locked state owned by that thread. When the next mapper thread calls `mr_emit_i` and tries to `pthread_mutex_lock(&g_inter_mu)`, it blocks indefinitely waiting for a lock that will never be released. This is a classic **deadlock** scenario. The entire program hangs.

</details>

---

## Section 4 — Function Pointers

---

**Q13.** Which of the following correctly declares a variable `fp` as a pointer to a function that takes a `const struct mr_in_kv *` parameter and returns `void`?

- A) `void fp*(const struct mr_in_kv *);`
- B) `(void) fp(const struct mr_in_kv *);`
- C) `void (*fp)(const struct mr_in_kv *);`
- D) `void *fp(const struct mr_in_kv *);`

<details>
<summary>Reveal Answer</summary>

**Correct Answer: C — `void (*fp)(const struct mr_in_kv *);`**

Reading the declaration from the variable name outward:
- `fp` — it's a variable
- `(*fp)` — it is a pointer
- `(*fp)(const struct mr_in_kv *)` — it is a pointer to a function taking that arg
- `void (*fp)(...)` — the function returns void

**D trap:** `void *fp(const struct mr_in_kv *)` declares `fp` as a **function** (not a pointer to a function) that returns `void *`. The parentheses around `*fp` in option C are essential — they group the `*` with `fp`, making it a pointer declaration rather than a function declaration.

</details>

---

**Q14.** In `mapreduce.c`, the `mapper_arg` struct stores the map function as:
```c
struct mapper_arg {
    const struct mr_in_kv *start;
    void (*map)(const struct mr_in_kv *);
    size_t count;
};
```
Inside `mapper_fn`, how is the stored function pointer actually called?

- A) `call a->map(&a->start[i]);`
- B) `invoke(a->map, &a->start[i]);`
- C) `a->map.call(&a->start[i]);`
- D) `a->map(&a->start[i]);`

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — `a->map(&a->start[i]);`**

In C, calling a function through a pointer uses the same syntax as a regular function call — just use the pointer variable name as if it were a function name:
```c
a->map(&a->start[i]);
// Equivalent explicit form:
(*a->map)(&a->start[i]);
```
Both are valid C. The simplified form (D) is more commonly written. There is no `call` keyword or `.call` method syntax in C.

</details>

---

**Q15.** A developer wants to pass their own custom word-count map function to `mr_exec`. Their function is:
```c
void my_map(const struct mr_in_kv *kv) { mr_emit_i(kv->value, "1"); }
```
Which call correctly passes it?

- A) `mr_exec(&input, &(&my_map), 1, my_reduce, 1, &output);`
- B) `mr_exec(&input, my_map, 1, my_reduce, 1, &output);`
- C) `mr_exec(&input, (void*)my_map, 1, my_reduce, 1, &output);`
- D) `mr_exec(&input, *my_map, 1, my_reduce, 1, &output);`

<details>
<summary>Reveal Answer</summary>

**Correct Answer: B — `mr_exec(&input, my_map, 1, my_reduce, 1, &output);`**

Function names in C **decay to function pointers** when used in an expression (other than as the operand of `&` or `sizeof`). So `my_map` and `&my_map` are both valid and equivalent ways to pass a function pointer. Option B uses the plain name — the simplest and most idiomatic form.

Option C would cause a type mismatch (casting to `void*` loses type information).
Option D — dereferencing a function pointer (`*my_map`) just gives back the function itself, which decays back to a pointer, so it technically works but is needlessly confusing and unusual.

</details>

---

## Section 5 — The Algorithm and Group-By-Key

---

**Q16.** In `single_reduce.c`, the input is constructed so that keys are `i % 8` (values 0–7) for 1024 input pairs. After the map stage passes everything through unchanged, how many times is `sr_reduce` called?

- A) 1024 — once per input pair
- B) 128 — once per 8-input group
- C) 1 — reduce is only called once for the entire run
- D) 8 — once per unique key (there are only 8 distinct keys: "0" through "7")

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — 8 times**

The group-by-key stage collects all 1024 intermediate pairs and groups them by key. Since keys are `i % 8`, there are exactly **8 unique keys** (`"0"` through `"7"`), each with 128 associated values. The reduce stage calls `sr_reduce` **once per grouped pair** — that is, once per unique key = **8 times**.

`sr_call_count == MOD` (where `MOD = 8`) is exactly what `single_reduce()` verifies.

</details>

---

**Q17.** Why does the framework sort `g_inter[]` by key before grouping, rather than using a hash table directly?

- A) Hash tables are not available in standard C
- B) Sorting is O(n log n) which is always faster than hash table O(n)
- C) Sorting makes the group-by-key stage a single O(n) linear scan where identical keys are already contiguous — no need for hash collisions or dynamic resizing; the sorted output also satisfies the requirement that mapped intermediate pairs are ordered alphabetically
- D) The assignment bans hash tables

<details>
<summary>Reveal Answer</summary>

**Correct Answer: C — Sorting enables a simple O(n) grouping pass and produces sorted output**

After sorting, all entries with the same key are contiguous. A single left-to-right scan can detect group boundaries (where key changes) and collect values — no lookups, no hash collisions, no resizing. The output is also **sorted by key**, which satisfies the assignment's requirement that intermediate and final outputs are ordered by `strcmp`. A hash table would require an additional sort pass to meet the ordering requirement, negating its lookup speed advantage.

</details>

---

**Q18.** In `map_and_reduce.c`, the `full_map_reduce()` test iterates through all combinations of `m ∈ {2,4,8,16,32}` mapper threads and `r ∈ {2,4,8,16,32}` reducer threads. How many total `mr_exec` calls does this test make?

- A) 10 (5 mapper values + 5 reducer values)
- B) 32 (16 + 16)
- C) 5 (one per power of 2)
- D) 25 (5 × 5 = all combinations)

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — 25 calls**

```c
for (size_t i = 0; i < 5; i++) {         // 5 values of m
    size_t m = 1 << (i + 1);             // m = 2, 4, 8, 16, 32
    for (size_t j = 0; j < 5; j++) {     // 5 values of r
        size_t r = 1 << (j + 1);         // r = 2, 4, 8, 16, 32
        mr_exec(..., m, ..., r, ...);
    }
}
```
5 outer iterations × 5 inner iterations = **25 calls to `mr_exec`**. Every call must produce the exact same output (57 unique words with correct counts) regardless of thread counts.

</details>

---

## Section 6 — Edge Cases and Implementation Details

---

**Q19.** `mr_emit_i` uses this doubling pattern for capacity management:
```c
g_inter_cap = g_inter_cap ? g_inter_cap * 2 : 64;
g_inter = realloc(g_inter, g_inter_cap * sizeof(struct mr_in_kv));
```
What would go wrong if the initial capacity were `1` instead of `64`?

- A) The program would crash immediately on the first emit call
- B) `realloc` does not support sizes smaller than 64 bytes
- C) Nothing — correctness is unaffected, though with 1024 inputs emitting intermediates there would be approximately 10 realloc calls instead of 4, causing more overhead from repeated memory reallocations as the array doubles from 1 → 2 → 4 → ... → 1024
- D) The array would never grow beyond 64 elements

<details>
<summary>Reveal Answer</summary>

**Correct Answer: C — More realloc calls but still correct**

Starting from capacity 1: doublings would be 1→2→4→8→16→32→64→128→256→512→1024 = **10 reallocations** instead of starting at 64 (64→128→256→512→1024 = **4 reallocations**). The code is still correct — `realloc` works fine with any starting size — but starting at 64 reduces the number of realloc calls for typical workloads, which is a performance optimization. The assignment's implementation uses 64 as a pragmatic initial value.

</details>

---

**Q20.** When `mr_exec` is called a second time (e.g., in the grader's `multiple_calls` test), what must happen to `g_inter` and `g_final` at the very start of the second call to ensure correct results?

- A) They must be copied to a backup buffer
- B) They can be left as-is because the index resets to 0, so old data will be overwritten
- C) They must be sorted before the new call begins
- D) They must be freed and reset to `NULL` with counts and capacities zeroed — otherwise intermediate and final results from the first call would contaminate the second call's data

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — Free and reset to NULL/0**

`g_inter` and `g_final` are **global dynamic arrays** that persist between function calls. If we only reset `g_inter_n = 0` without freeing and setting to NULL:
- The old data from call 1 would still be in memory
- New map threads would write at index 0 onwards, overwriting old data — but the **capacity** would still reflect old allocations
- Crucially: forgetting to `free` the old allocation causes a **memory leak**

The correct approach (as in our code):
```c
free(g_inter);   // release old memory
g_inter = NULL;  // prevent dangling pointer use
g_inter_n = 0;
g_inter_cap = 0; // must also reset cap so next realloc starts fresh
```
Without resetting `g_inter_cap`, the `realloc` would use a stale capacity value pointing to already-freed memory — undefined behavior.

</details>

---

## Bonus — Tricky Syntax

---

**Q21 (Bonus).** What is the size of the memory block allocated by the following expression in `mr_exec`?
```c
grouped[gi].value = malloc(run * sizeof(char[MAX_VALUE_SIZE]));
```
If `run = 3` and `MAX_VALUE_SIZE = 16`, how many bytes are allocated?

- A) 3 bytes (sizeof char is 1, times 3)
- B) 24 bytes (3 × 8 pointer bytes)
- C) 16 bytes (one MAX_VALUE_SIZE block)
- D) 48 bytes (3 strings × 16 bytes each)

<details>
<summary>Reveal Answer</summary>

**Correct Answer: D — 48 bytes**

`sizeof(char[MAX_VALUE_SIZE])` = `sizeof(char[16])` = **16 bytes** (an array of 16 chars). Multiplied by `run = 3`: `3 × 16 = 48 bytes`. This allocates a flat block for 3 strings, each 16 bytes long. Accessing the i-th string: `grouped[gi].value[i]` is a `char[16]` at byte offset `i * 16`.

</details>

---

**Q22 (Bonus).** The `cmp_in_kv` comparator is also used on `g_final[]`, which is typed as `struct mr_in_kv *`. Yet `g_final` stores the final *output* key-value pairs. Why does this work even though final outputs are conceptually different from intermediate inputs?

- A) It is a bug — a different comparator should be used for final output
- B) The C standard allows using any comparator with any array type
- C) Both `g_inter` and `g_final` use `struct mr_in_kv` as their element type (both are flat string-pair arrays), and the comparator only inspects the `key` field — so the comparison logic is identical and correct for both
- D) `qsort` ignores the comparator when the array is already partially sorted

<details>
<summary>Reveal Answer</summary>

**Correct Answer: C — Same struct type, same key field layout**

Despite being semantically different (intermediate vs. final), both `g_inter` and `g_final` are declared as `struct mr_in_kv *` arrays. `mr_emit_f` stores final pairs using `strncpy` into `.key` and `.value` fields — the same fields that `cmp_in_kv` compares. Since the struct type and the field being compared are identical, reusing the comparator is correct and intentional. This avoids writing a duplicate comparator function.

</details>

---

*End of quiz. Check your answers, then review any concepts you missed in `concepts_learning_outcomes.md` or `code_implementation_recap.md`.*
