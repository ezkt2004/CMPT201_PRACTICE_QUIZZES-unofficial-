# 🧪 A8: Shell — MCQ Practice Bank (Quiz 3 Prep)

> **Coverage:** All A8 Learning Outcomes — C Skills, Process Management, Linux Programming, Signals, CMake/GTest
> **Difficulty:** Medium | **Total:** 27 Questions
> Click the **▶ Answer** toggle under each question to reveal the answer + explanation.

---

## 🔤 C Skills

---

**Q1.** You call `strtok_r(input, " \n", &saveptr)` and then pass `input` to another function that needs the original string. What is the problem?

- A) `strtok_r()` returns `NULL` if there are no spaces
- B) `saveptr` must be initialized to `input` before the first call
- C) `strtok_r()` modifies the original string by replacing delimiters with `'\0'`
- D) `strtok_r()` is not re-entrant and will corrupt the token state

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

`strtok_r()` **destructively modifies** the input string — it replaces delimiters (spaces, newlines) with null bytes `'\0'` to terminate each token. This is why in A8's `main.c`, a `raw_input` copy is saved **before** calling `tokenize_input()`, so the original command string can be added to history intact.

```c
char raw_input[512];
strncpy(raw_input, input, 511);  // save BEFORE tokenize destroys it
raw_input[511] = '\0';
tokenize_input(input, args, 64); // this mangles input
add_to_history(raw_input);       // use the clean copy
```

</details>

---

**Q2.** What is the key difference between `strtok()` and `strtok_r()`?

- A) `strtok_r()` is faster because it avoids heap allocation
- B) `strtok_r()` takes an explicit `saveptr` argument making it re-entrant and thread-safe
- C) `strtok()` supports regex delimiters; `strtok_r()` only supports single characters
- D) `strtok_r()` does not modify the input string

<details>
<summary>▶ Answer</summary>

**Correct Answer: B**

`strtok()` stores its position in a hidden **static variable**, making it unsuitable for concurrent or nested use (not re-entrant). `strtok_r()` takes a `char **saveptr` argument that the caller manages, making it **re-entrant** — safe for use in signal handlers, nested loops, or threaded code.

```c
// strtok_r signature:
char *strtok_r(char *str, const char *delim, char **saveptr);
//                                                  ^^^^^^^^ caller-owned state
```

</details>

---

**Q3.** In A8's `tokenize_input()`, after the `while` loop finishes, the code does `arg[i] = NULL`. Why is this critical?

- A) To signal to `strtok_r()` that tokenization is complete
- B) To prevent `strcmp()` from reading past the last argument
- C) Because `memset(args, 0, sizeof(args))` in `main` already handles this and it's redundant
- D) Because `execvp()` requires the argument array to be null-terminated

<details>
<summary>▶ Answer</summary>

**Correct Answer: D**

`execvp(path, argv)` expects `argv` to be a **null-terminated array of strings** — it uses the `NULL` sentinel to know when the list ends. Without `arg[i] = NULL`, `execvp()` would read garbage memory past the last valid argument, causing undefined behavior or crashes. This is a POSIX requirement.

```c
// execvp prototype:
int execvp(const char *file, char *const argv[]);
// argv[last] MUST be NULL
```

</details>

---

**Q4.** You need to safely build a formatted error message and output it. Which approach is both correct and signal-safe?

- A) `printf("cd: %s\n", TMA_MSG);`
- B) `fprintf(stderr, FORMAT_MSG("cd", TMA_MSG));`
- C) Build the string with `snprintf()` outside a signal handler, then use `write()` to output it
- D) Use `puts()` with the pre-formatted string constant

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

`printf()`, `fprintf()`, and `puts()` are **not async-signal-safe** (they use internal locks and buffered I/O). The correct A8 pattern is:

```c
char error_message[128];
snprintf(error_message, sizeof(error_message), FORMAT_MSG("cd", TMA_MSG));
write(STDERR_FILENO, error_message, strlen(error_message));
```

Inside a **signal handler**, even `snprintf()` is technically not signal-safe. A8 solves this by pre-computing strings at startup (`init_help_messages_signal_safe()`) and only using `write()` inside the handler.

</details>

---

**Q5.** What does `strncpy(dest, src, n)` guarantee that `strcpy(dest, src)` does NOT?

- A) It always appends a null terminator at `dest[n]`
- B) It copies at most `n` bytes, preventing writes beyond the destination buffer
- C) It verifies that `src` is a valid null-terminated string before copying
- D) It is signal-safe and can be called from within signal handlers

<details>
<summary>▶ Answer</summary>

**Correct Answer: B**

`strncpy(dest, src, n)` copies **at most `n` bytes**, protecting against buffer overflow. However — and this is a classic trap — if `src` is longer than `n`, `strncpy` does **NOT** add a null terminator. A8 handles this explicitly:

```c
strncpy(history[index], input, 511);
history[index][511] = '\0';  // manually force null termination!
```

`strcpy()` has no length limit and will overflow the buffer if `src` is too long.

</details>

---

**Q6.** The history array in A8 is declared as `static char history[HISTORY_SIZE][512]`. If `history_count = 23` and `HISTORY_SIZE = 10`, what index does the **next** command get stored at?

- A) `history[3]` — index 3
- B) `history[10]` — out of bounds
- C) `history[0]` — wraps to beginning
- D) `history[2]` — index 2, overwriting command #12

> Hint: index = `history_count % HISTORY_SIZE`

<details>
<summary>▶ Answer</summary>

**Correct Answer: A**

The ring buffer index is `history_count % HISTORY_SIZE` = `23 % 10` = **3**. This overwrites whatever was at index 3 (which was command #3 from the beginning). The 10 currently stored commands in a full buffer are at indices `history_count % 10` through `(history_count - 1) % 10` cyclically.

```
Command #23 → index 3 (overwrites command #13, which had overwritten #3)
```

</details>

---

**Q7.** A user has entered exactly 13 commands (numbered 0–12). They run `history`. Which commands are displayed and in what order?

- A) Commands 0–12 in ascending order (13 entries shown)
- B) Commands 3–12 in descending order (10 entries, most recent first)
- C) Commands 3–12 in ascending order (10 entries, oldest first)
- D) Commands 0–9 in descending order (first 10 commands)

<details>
<summary>▶ Answer</summary>

**Correct Answer: B**

History always shows the **most recent 10** in **descending order** (newest first). With `history_count = 13`:
- Valid range: commands `3` through `12` (last 10)
- Display: `12, 11, 10, 9, 8, 7, 6, 5, 4, 3`

```c
int count = (history_count < HISTORY_SIZE) ? history_count : HISTORY_SIZE;
// count = 10
for (int i = history_count - 1; i >= history_count - count; i--) {
    // i = 12, 11, 10, ..., 3
    int index = i % HISTORY_SIZE;
    // ...
}
```

</details>

---

## ⚙️ Process Management

---

**Q8.** After calling `fork()`, the parent process receives the return value `4821`. What does this mean?

- A) `4821` is the process ID of the child; the parent should now call `waitpid(4821, ...)` for foreground
- B) `4821` indicates an error — `fork()` failed and `errno` is set
- C) `4821` is the parent's own PID being reconfirmed; the child received `0`
- D) `4821` means the child exited immediately with status 4821

<details>
<summary>▶ Answer</summary>

**Correct Answer: A**

`fork()` return value semantics:
- `< 0` → **error**, fork failed
- `== 0` → you are the **child** process
- `> 0` → you are the **parent**, and the value is the **child's PID**

So `4821` means the parent got back the child's PID. For foreground execution, A8 then does:
```c
waitpid(pid, &wstatus, 0);  // wait for THIS specific child
```

</details>

---

**Q9.** In `execute_command()`, when `execvp()` is called in the child process and it **succeeds**, what happens next?

- A) `execvp()` returns 0 and the child continues executing the shell code below it
- B) `execvp()` returns the new program's PID so the parent can wait on it
- C) The child process's memory image is replaced — the code after `execvp()` is never reached
- D) The child sleeps until the parent calls `waitpid()`, then wakes up and executes

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

When `execvp()` **succeeds**, it **replaces** the current process's entire memory image (code, data, stack, heap) with the new program. The function never returns to the caller. This is why the error-printing code after `execvp()` only runs if exec **fails**:

```c
execvp(arg[0], arg);
// If we reach here, execvp FAILED:
write(STDERR_FILENO, "shell: unable to execute command\n", ...);
exit(EXIT_FAILURE);
```

</details>

---

**Q10.** What is a **zombie process** and when does one appear in A8?

- A) A child process that calls `exit()` before the parent calls `fork()`
- B) A process that exceeds its memory limit and is killed by `SIGKILL`
- C) A child that has been running in the background for more than 60 seconds
- D) A child that has terminated but whose parent hasn't called `wait()` yet, leaving its exit status in the process table

<details>
<summary>▶ Answer</summary>

**Correct Answer: D**

A zombie process has **finished executing** but its entry remains in the OS process table until the parent calls `wait()`/`waitpid()` to collect the exit status. In A8, background processes (`sleep 5 &`) become zombies when they finish because the parent shell doesn't immediately wait on them. The fix is `cleanup_zombie_processes()` called after every command:

```c
while ((pid = waitpid(-1, &wstatus, WNOHANG)) > 0) {
    // zombie reaped — nothing else needed
}
```

</details>

---

**Q11.** Which `waitpid()` call correctly cleans up zombie background processes without blocking the shell?

- A) `waitpid(0, &status, 0)` — wait for any child in the same process group, blocking
- B) `waitpid(-1, &status, 0)` — wait for any child, blocking until one exits
- C) `waitpid(pid, &status, WNOHANG)` — wait for specific child, non-blocking
- D) `waitpid(-1, &status, WNOHANG)` — wait for any child, returning immediately if none have exited

<details>
<summary>▶ Answer</summary>

**Correct Answer: D**

The two key arguments:
- `pid = -1` → wait on **any** child (not a specific one, since we may have many background processes)
- `WNOHANG` → **non-blocking** — return 0 immediately if no child has exited yet

These must be combined. Option B would block the shell until a background process exits. Option C waits for a specific PID (fine for foreground, wrong for zombie cleanup). This loop is called after **every** command:

```c
while ((pid = waitpid(-1, &wstatus, WNOHANG)) > 0) { /* reap */ }
```

</details>

---

**Q12.** After the zombie cleanup loop, you check `if (pid == -1 && errno != ECHILD)`. Why is `errno != ECHILD` important here?

- A) `ECHILD` means "no children exist at all" — this is a normal exit from the loop, not an error worth reporting
- B) `ECHILD` means the child was killed by a signal and its status was lost
- C) `ECHILD` is set when `WNOHANG` caused an early return — it must be re-tried
- D) `ECHILD` indicates a memory corruption in the process table that needs to be logged

<details>
<summary>▶ Answer</summary>

**Correct Answer: A**

When `waitpid(-1, ...)` returns `-1`, it could mean:
- `errno == ECHILD` → **no children left to wait on** → loop correctly exhausted all zombies → **not an error**
- `errno == something_else` → a real unexpected error occurred → report it

Without checking `errno != ECHILD`, you'd print "unable to wait for child" every time the cleanup loop finishes cleanly, which would spam the terminal after every command.

</details>

---

**Q13.** In A8's `check_background_execution()`, after detecting `&` as the last argument, the code sets `arg[i-1] = NULL`. Why?

- A) To trigger `WNOHANG` mode in the following `waitpid()` call
- B) To prevent `strcmp()` from comparing `&` with internal command names
- C) Because `fork()` refuses to execute if `&` appears in the argument list
- D) To remove `&` from the args array so `execvp()` doesn't pass it as an argument to the program being run

<details>
<summary>▶ Answer</summary>

**Correct Answer: D**

`execvp()` passes the `args[]` array directly to the new process. If `&` wasn't removed, `sleep` would be called as `sleep 10 &` — the `&` would be passed as an argument TO `sleep`, which would cause it to fail. Setting `arg[i-1] = NULL` effectively truncates the array, removing `&` before execution.

</details>

---

## 🐧 Linux Programming

---

**Q14.** The user types `cd ~/projects/cmpt201`. How does A8 resolve `~` to the actual path?

- A) The shell calls `glob()` to expand `~` using the filesystem
- B) `chdir("~")` is called directly — the kernel resolves `~` automatically
- C) The shell calls `getpwuid(getuid())` to look up the home directory from `/etc/passwd`
- D) `getenv("HOME")` is used, then concatenated with `arg[1]+1` (the path after `~`)

<details>
<summary>▶ Answer</summary>

**Correct Answer: D**

A8's `cd_cmd()` expands `~` manually:

```c
const char *home = getenv("HOME");
if (home != NULL) {
    snprintf(target_buf, sizeof(target_buf), "%s%s", home, arg[1] + 1);
    //                                        ^^^^   ^^^^^^^^^^^
    //                                     /home/user  /projects/cmpt201
    target = target_buf;
}
```

`arg[1] + 1` is **pointer arithmetic** that skips the `~` character. So `~/projects` becomes `/home/user/projects`. Note: `getpwuid()` is an alternative approach but is **not required** to memorize for the quiz.

</details>

---

**Q15.** A user does: `cd /tmp`, then `cd /var`, then `cd -`. What does `cd -` do and what is the final CWD?

- A) Changes to the home directory (`$HOME`) — `cd -` is an alias for `cd ~`
- B) Moves up one directory level — equivalent to `cd ..`
- C) Changes back to `/tmp` (the directory before the last `cd`)
- D) Prints an error: `cd: unable to change directory` because `-` is not a valid path

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

`cd -` changes to `prev_dir`, which is saved **before each successful `cd`**. After `cd /var`, `prev_dir = "/tmp"`. So `cd -` calls `chdir("/tmp")`, making CWD `/tmp`.

```c
// In handle_internal(), before calling cd_cmd():
char old_cwd[PATH_MAX];
getcwd(old_cwd, sizeof(old_cwd));  // save current dir
int result = cd_cmd(arg);
if (result == 0) {
    strncpy(prev_dir, old_cwd, PATH_MAX - 1);  // update prev_dir
    init_prompt_signal_safe();                  // update signal-safe prompt
}
```

</details>

---

**Q16.** `getcwd()` is called and returns `NULL`. What does A8's shell do?

- A) Silently retries with a larger buffer up to 3 times before giving up
- B) Exits the shell process since it cannot determine its location
- C) Prints `shell: unable to get current directory` to stderr and continues
- D) Falls back to using `getenv("PWD")` as an alternative

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

A8 uses the `GETCWD_ERROR_MSG` macro for this case. The shell continues running — it doesn't crash. From `display_prompt()`:

```c
cwd = getcwd(buf, sizeof(buf));
if (cwd != NULL) {
    // display normal prompt
} else {
    char error_message[128];
    snprintf(error_message, sizeof(error_message),
             FORMAT_MSG("shell", GETCWD_ERROR_MSG));
    write(STDERR_FILENO, error_message, strlen(error_message));
}
```

Output: `shell: unable to get current directory`

</details>

---

## 📡 Signals

---

**Q17.** Which function is the correct way to register a signal handler in A8, and why is it preferred?

- A) `signal(SIGINT, handler)` — simpler API, resets to default after first delivery
- B) `sigaction(SIGINT, &sa, NULL)` — more portable, allows control of handler flags and signal masking
- C) `raise(SIGINT)` — sends the signal immediately to test the handler
- D) `kill(getpid(), SIGINT)` — registers and immediately tests the handler

<details>
<summary>▶ Answer</summary>

**Correct Answer: B**

`sigaction()` is the POSIX-preferred method because:
- It does **not** reset to default after one delivery (unlike older `signal()` implementations)
- Allows setting `sa_mask` to block other signals during handler execution
- The `sa_flags` field provides fine-grained control

```c
struct sigaction sa;
sa.sa_handler = signal_handler;
sigemptyset(&sa.sa_mask);   // don't block any additional signals
sa.sa_flags = 0;
sigaction(SIGINT, &sa, NULL);
```

</details>

---

**Q18.** Inside a signal handler, which of the following sets of operations is FULLY async-signal-safe?

- A) `printf()`, `strlen()`, `write()`
- B) `malloc()`, `write()`, `getpid()`
- C) `printf()`, `snprintf()`, `free()`
- D) `write()`, `strlen()`, `_exit()`

<details>
<summary>▶ Answer</summary>

**Correct Answer: D**

From `man signal-safety`:
- `write()` ✅ — signal-safe (direct syscall)
- `strlen()` ✅ — signal-safe (simple memory read, no locks)
- `_exit()` ✅ — signal-safe (does not flush stdio buffers)
- `printf()` ❌ — uses `FILE*` buffer with internal mutex
- `malloc()` / `free()` ❌ — use internal heap locks
- `snprintf()` ❌ — uses locale data and internal state

A8's design pre-computes strings at startup so the handler only needs `write()`:
```c
void signal_handler(int sig_number) {
    if (sig_number == SIGINT) {
        display_help_signal_safe();   // only write() calls inside
        display_prompt_signal_safe(); // only write() calls inside
    }
}
```

</details>

---

**Q19.** The user presses `Ctrl-C` while the shell is blocked in `read()`. The `read()` call returns `-1`. What should the main loop do?

- A) Print `shell: unable to read command` and exit the shell
- B) Print `shell: unable to read command` and continue to the next iteration
- C) Check `errno`: if `errno == EINTR`, just `continue` the loop; otherwise print the error
- D) Call `read()` again immediately without checking `errno`

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

`EINTR` (Interrupted system call) is not a real error — it means a signal interrupted the blocking call. The signal handler already ran and re-displayed the prompt. The correct response is to `continue` back to the top of the loop (which will call `display_prompt()` again, but the handler already did it). Any other `errno` value IS a real error.

```c
if (input_read == -1) {
    if (errno == EINTR) {
        continue;  // signal handled, just restart the loop
    } else {
        // real error - report it
        write(STDERR_FILENO, error_msg, strlen(error_msg));
        continue;
    }
}
```

</details>

---

**Q20.** Why does A8 call `init_help_messages_signal_safe()` and `init_prompt_signal_safe()` at **startup** before the main loop, rather than building these strings inside the signal handler itself?

- A) These functions use `malloc()` which can only be called once per program
- B) `snprintf()` is not async-signal-safe, so strings must be pre-built outside the handler; the handler only uses `write()` with pre-computed buffers and lengths
- C) The signal handler runs in a separate thread where stack space is limited
- D) `SIGINT` can arrive before `main()` starts, so initialization must happen in a constructor

<details>
<summary>▶ Answer</summary>

**Correct Answer: B**

`snprintf()` is NOT async-signal-safe (it uses locale data and internal state). By building all strings **before** any signal can arrive, the handler only needs to call `write()` with pre-computed char arrays and pre-computed lengths (avoiding even `strlen()` for maximum safety):

```c
// Startup (safe to use snprintf here):
snprintf(exit_help, sizeof(exit_help), FORMAT_MSG("exit", EXIT_HELP_MSG));
exit_help_len = strlen(exit_help);  // pre-compute length too

// Inside signal handler (only write() — fully signal-safe):
write(STDOUT_FILENO, exit_help, exit_help_len);
```

</details>

---

## 📜 History & Bang Commands

---

**Q21.** A user enters `!!` but `history_count == 0`. What should happen?

- A) The shell re-runs nothing and silently returns to the prompt
- B) The shell exits because there is no valid state to re-run
- C) The shell prints `history: no command entered` to stderr and the history stays empty
- D) The shell prints `history: command invalid` and adds `!!` to the history

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

The `HISTORY_NO_LAST_MSG` macro is used: `"no command entered"`. Crucially:
- The `!!` itself is **NOT** added to history
- The history remains empty
- The error message format is `history: no command entered`

```c
if (history_count == 0) {
    char error_message[128];
    snprintf(error_message, sizeof(error_message),
             FORMAT_MSG("history", HISTORY_NO_LAST_MSG));
    write(STDERR_FILENO, error_message, strlen(error_message));
    return 1;  // failure: main loop will 'continue' without executing
}
```

</details>

---

**Q22.** The user has entered 15 commands (numbered 0–14). They type `!3`. What happens?

- A) The shell re-runs command #3 because it is within the last 10 (commands 5–14)
- B) The shell prints `history: command invalid` because command #3 is too old (not in the last 10)
- C) The shell re-runs the 3rd most recent command (which is command #11)
- D) The shell prints `history: command invalid` because `!3` is a single-digit command reference

<details>
<summary>▶ Answer</summary>

**Correct Answer: B**

With `history_count = 15`, the valid range of commands accessible via `!n` is:
- Minimum: `history_count - HISTORY_SIZE` = `15 - 10` = **5**
- Maximum: `history_count - 1` = **14**

Command #3 is **outside this range**, so the shell prints `history: command invalid`. Only `!5` through `!14` would be valid.

```c
if (n < history_count - HISTORY_SIZE || n >= history_count) {
    // error: "history: command invalid"
}
```

</details>

---

**Q23.** When a command is re-run via `!!` or `!n`, which of the following correctly describes what gets added to the history?

- A) The literal string `!!` or `!n` is added (e.g., `!!` becomes a history entry)
- B) Nothing is added to history when using `!` commands — history is read-only during re-runs
- C) The command that was re-run from history is added (not `!!` / `!n`), and it's also printed to stdout before execution
- D) Both `!!` and the command it re-ran are added as two separate history entries

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

Per the A8 README spec and implementation:
- `!!` / `!n` themselves are **NOT** added to history
- The **original command being re-run** IS added to history
- The command is **printed to stdout** before execution (so the user sees what's running)

```c
// In bang_cmd():
write(STDOUT_FILENO, history[index], strlen(history[index]));  // display it
write(STDOUT_FILENO, "\n", 1);
// ...
add_to_history(temp);  // add the original command to history
```

</details>

---

**Q24.** The user types `!abc` (a non-numeric argument after `!`). What should the shell do?

- A) Treat it as a regular external command named `!abc` and try to execute it
- B) Print `history: no command entered` since the argument is not a valid number
- C) Print `history: command invalid` because `abc` is not a valid history number
- D) Print `shell: unable to execute command` after failing to run it as an external program

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

A8 uses `strtol()` to parse the number after `!`. The `endptr` check determines if the entire string was a valid integer:

```c
char *endptr;
long n = strtol(arg[0] + 1, &endptr, 10);  // arg[0]+1 skips the '!'

if (*endptr != '\0') {
    // endptr didn't reach end of string → not a pure number → invalid
    // print: "history: command invalid"
}
```

If `arg[0]` is `"!abc"`, `strtol("abc", &endptr, 10)` returns `0` with `*endptr == 'a'` (not `'\0'`), triggering the invalid case.

</details>

---

## 🔨 CMake & GTest

---

**Q25.** You run `cmake --build build` but get an error: `make[2]: *** No rule to make target 'shell'`. What is the most likely cause?

- A) The `clang` compiler is not installed on your system
- B) You forgot to run `cmake -S . -B build` first to generate the build system
- C) Your `CMakeLists.txt` has `add_executable(shell_test ...)` but is missing `add_executable(shell ...)` for the shell itself
- D) GTest is not installed, preventing compilation of all targets

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

The GTest test suite (`shell_test.cpp`) **expects an executable named `shell` in the build directory**. If you only define `shell_test` in `CMakeLists.txt` but forget to declare the `shell` target, the shell binary won't be compiled and the tests will fail.

```cmake
# Both targets are required:
add_executable(shell              # ← produces ./build/shell
    src/main.c
    src/external_commands.c
    src/internal_commands.c
)

add_executable(shell_test         # ← produces ./build/shell_test
    gtest/shell_test.cpp
    src/external_commands.c
    src/internal_commands.c
)
```

</details>

---

**Q26.** You see this GTest output line: `[  FAILED  ] ShellTest.Pwd (500 ms)`. What is the most useful next step?

- A) Recompile with `-O2` optimization — test failures are often caused by unoptimized code
- B) Run `shell_test --gtest_filter=ShellTest.Pwd` to re-run just that test, then manually run `./build/shell` and type `pwd` to see the actual output and compare
- C) Delete the build directory and rerun `cmake` — the failure is likely a stale build artifact
- D) Add `#define NDEBUG` to `internal_commands.c` to disable debug assertions

<details>
<summary>▶ Answer</summary>

**Correct Answer: B**

The A8 README explicitly says: *"If a test case fails for you, go to the test case source, find out the input it uses and the output it expects, and start your own debugging process with the input."* The right workflow is:
1. Run the specific failing test (or read its source in `shell_test.cpp`)
2. Find what input it sends and what output it expects
3. Manually test your shell with that exact input
4. Compare actual vs expected output character-by-character (watch for invisible chars like `\n` vs `\r\n`)

</details>

---

**Q27.** Which CMake commands and in what order correctly set up the A8 build system from scratch?

- A) `cmake --build build` → `cmake -S . -B build` → `make`
- B) `make -S . -B build` → `cmake --build build`
- C) `export CC=$(which clang) ; export CXX=$(which clang++) ; cmake -S . -B build ; cmake --build build`
- D) `clang -o build/shell src/*.c ; clang++ -o build/shell_test gtest/*.cpp`

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

The full correct sequence is:
```bash
export CC=$(which clang)      # tell CMake to use clang for C
export CXX=$(which clang++)   # tell CMake to use clang++ for C++
cmake -S . -B build           # configure: generate build system in ./build/
cmake --build build           # build: compile everything
./build/shell                 # run your shell
./build/shell_test            # run the test suite
```

`CC` and `CXX` must be set **before** `cmake -S . -B build`, not after. Option D would work but bypasses CMake entirely (which is required for grading).

</details>

---

## 🎯 Mixed / Tricky

---

**Q28.** All of the following code snippets involve A8's `FORMAT_MSG` macro. Which one is used correctly?

- A) `write(STDERR_FILENO, FORMAT_MSG("exit", TMA_MSG), strlen(FORMAT_MSG("exit", TMA_MSG)));`
- B) `char buf[128]; snprintf(buf, sizeof(buf), FORMAT_MSG("exit", TMA_MSG)); write(STDERR_FILENO, buf, strlen(buf));`
- C) `printf(FORMAT_MSG("exit", TMA_MSG));`
- D) `puts(FORMAT_MSG("exit", TMA_MSG));`

<details>
<summary>▶ Answer</summary>

**Correct Answer: B**

`FORMAT_MSG` expands to a **string literal** like `"exit: too many arguments\n"`. It can be used directly as a `printf` format string (but `printf` isn't signal-safe). The A8 pattern uses `snprintf()` to build into a buffer, then `write()` to output it:

```c
char error_message[128];
snprintf(error_message, sizeof(error_message), FORMAT_MSG("exit", TMA_MSG));
// expands to: snprintf(buf, 128, "exit: " "too many arguments" "\n");
write(STDERR_FILENO, error_message, strlen(error_message));
```

Option A would call `FORMAT_MSG` twice (two separate `strlen` evaluations of a literal — technically works but terrible style). Options C/D use non-signal-safe functions and send to wrong stream.

</details>

---

**Q29.** In the main loop, `memset(args, 0, sizeof(args))` is called at the **start** of each iteration. Why is this important?

- A) To reset the `strtok_r` saveptr from the previous iteration
- B) To prevent `tokenize_input()` from being called with uninitialized memory
- C) To ensure leftover `args` pointers from a previous command don't accidentally get passed to `execvp()` or `handle_internal()` if the new command has fewer arguments
- D) To zero out the `input` buffer so old input characters don't corrupt the new command

<details>
<summary>▶ Answer</summary>

**Correct Answer: C**

Consider: previous command was `ls -la -h` (fills `args[0..3]`). Next command is `pwd` (fills only `args[0..1]`). Without `memset`, `args[2]` still points to `-h` from the old iteration. `handle_internal()` checks `arg[1]` for `pwd` and would incorrectly think there's an argument. The `memset` ensures every unused slot is `NULL` from the start.

</details>

---

**Q30.** The README states: *"For foreground execution, you need to wait for the exact process you fork and not for any other processes."* Which call implements this correctly?

- A) `waitpid(-1, &wstatus, 0)` — waits for any child that exits (blocking)
- B) `waitpid(-1, &wstatus, WNOHANG)` — waits for any child non-blocking
- C) `wait(&wstatus)` — waits for any child (equivalent to `waitpid(-1, &wstatus, 0)`)
- D) `waitpid(pid, &wstatus, 0)` — waits specifically for the child with PID `pid` (blocking)

<details>
<summary>▶ Answer</summary>

**Correct Answer: D**

For foreground execution, you must wait for the **specific child** you just forked — not just "any" child. Using `waitpid(-1, ...)` or `wait()` could accidentally reap a background child process instead of the foreground one, causing the shell to return to the prompt before the foreground command finishes.

```c
pid_t pid = fork();
// ... in parent:
int wstatus;
waitpid(pid, &wstatus, 0);  // wait for THIS pid, block until done
```

</details>

---

*Good luck on Quiz 3! 🎓 Remember: understand the WHY behind each answer, not just the answer letter.*
