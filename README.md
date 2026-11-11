# Custom Linux Shell & System Call Utilities (`CS39002`)

**Course:** CS39002 — Operating Systems Laboratory (Spring 2022–2023, IIT Kharagpur)  
**Assignment:** Assignment 2 — Use of System Calls (Custom Unix Shell Design)  
**Problem Statement:** [`OSlab-Asgn2-shell-design-2023-part-1-part-2.pdf`](./OSlab-Asgn2-shell-design-2023-part-1-part-2.pdf)  
**Malware Heuristic Report:** [`assgn2.squashbug.heuristic.txt`](./assgn2.squashbug.heuristic.txt)

---

## Overview

This repository contains a custom interactive **Linux Shell (`terminal`)** implemented in C/C++ on top of POSIX/Linux system calls and the **GNU Readline** library. In addition to standard shell capabilities—such as external command execution, I/O redirection (`<`, `>`), arbitrary-length command pipelines (`|`), background job execution (`&`), wildcard globbing (`*`, `?`), signal handling (`SIGINT`, `SIGTSTP`, `SIGCHLD`), and persistent 1,000-command history—the shell implements two specialized `/proc`-based kernel inspection utilities:
1. **`sb` (*squashbug*):** A heuristic trojan/malware detector that traverses the process ancestry tree via `/proc/<pid>/stat` and identifies sleeper-spawner malware processes based on child-spawning rate and recursive subtree CPU utilization.
2. **`delep` (*delete with extreme prejudice*):** A locked-file unlocker and deleter that spawns an inspection child process, communicates over an anonymous `pipe()`, scans `/proc/<pid>/fd/` and `/proc/locks` to identify processes holding open descriptors or `flock` locks on a target file, and terminates them with `SIGKILL` upon user confirmation before unlinking the file.

---

## Repository Structure

```text
.
├── README.md                                        # Project documentation (this file)
├── main.cpp                                         # Core shell REPL, signal handlers, parser dispatch, pipes & redirection
├── utility.hpp                                      # Data structures, constants, and function declarations
├── utility.cpp                                      # Implementations of history, wildcards, readline bindings, `sb`, and `delep`
├── makefile                                         # Build configuration for compiling `terminal`
├── malware.c                                        # Test payload for `sb`: spawns 5 children x 10 grandchildren in infinite loops
├── lock.cpp                                         # Test payload for `delep`: acquires an exclusive `flock` (LOCK_EX) in an infinite loop
├── assgn2.squashbug.heuristic.txt                   # Explanation and justification of the `sb -suggest` malware heuristic
└── OSlab-Asgn2-shell-design-2023-part-1-part-2.pdf  # Official assignment specification
```

---

## Build & Execution

### Prerequisites
- `g++` / `gcc` (C++17 or later)
- GNU Readline development library (`libreadline-dev` on Debian/Ubuntu)

```bash
sudo apt-get install build-essential libreadline-dev
```

### Compile and Run the Shell
```bash
make
./terminal
```

To clean the compiled binary:
```bash
make clean
```

---

## Implemented Features & Architecture

### Part 1: Core Shell & Process Control

| Feature | Syntax / Trigger | System Calls & Implementation Details |
| :--- | :--- | :--- |
| **External Command Execution** | `./a.out`, `ls -l`, `cc -o prog prog.c` | Spawns a child process via `fork()` and executes the binary via `execvp()`. Foreground jobs block the shell via `waitpid(childPid, NULL, 0)`. |
| **Input Redirection** | `cmd < infile.txt` | Opens `infile.txt` with `open(..., O_RDONLY)` and duplicates its file descriptor onto `STDIN_FILENO` (`0`) using `dup2(fd, 0)` followed by `close(fd)`. |
| **Output Redirection** | `cmd > outfile.txt` | Opens/creates `outfile.txt` with `open(..., O_RDWR \| O_CREAT, 0666)` and redirects `STDOUT_FILENO` (`1`) via `dup2(fd, 1)`. |
| **Combined I/O Redirection** | `./a.out < in.txt > out.txt` | Chains `dup2(fd_in, 0)` and `dup2(fd_out, 1)` prior to calling `execvp()`. |
| **Background Execution** | `./a.out &`, `./prog < in > out &` | Detects trailing `&` during tokenization (`split()`) and uses non-blocking `waitpid(childPid, NULL, WNOHANG)` with a `SIGCHLD` handler (`sigchld_handler`). |
| **Multi-Stage Pipelines** | `cmd1 \| cmd2 \| cmd3` | Iteratively creates anonymous pipes via `pipe(pipe_fd)`, forks child stages redirecting `STDOUT` (`dup2(pipe_fd[1], 1)`) of stage $i$ to `STDIN` (`dup2(pipe_fd[0], 0)`) of stage $i+1$, and manages alternating descriptor parity (`prev_pipe_fd`). |
| **Signal Handling (`Ctrl+C`, `Ctrl+Z`)** | `SIGINT`, `SIGTSTP`, `SIGCHLD` | Registered via `signal()` and `siginterrupt()`. Pressing `Ctrl+C` (`sigint_handler`) sends `SIGINT` to the active foreground process group (`killpg(getpgid(childPid), SIGINT)`) without terminating the shell. Pressing `Ctrl+Z` (`sigtstp_handler`) transitions the active process to the background and refreshes the prompt. |

---

### Part 2: Built-ins, Wildcards, History & Kernel Inspection Tools

#### 1. Directory Navigation (`cd`, `pwd`) & Prompt
- The shell prompt dynamically displays the current working directory via `getcwd()` (`<cwd> [#] `).
- Built-in `cd <path>` directly invokes the `chdir()` system call in the parent shell process.

#### 2. Wildcard Expansion (`*` and `?`)
- Implemented in `wildcard_split()` and `wildcard_handler()` ([`utility.cpp`](./utility.cpp)).
- Splits path patterns across `/` delimiters, traverses directories using `opendir()` and `readdir()`, and matches candidate entries against `*` and `?` patterns using `fnmatch(..., FNM_CASEFOLD)` before expanding matched paths into the argument vector.

#### 3. Persistent Command History & Line Editing
- Maintains a persistent double-ended queue (`std::deque<string>`) of up to **1,000 commands** backed by `.cmd_history` (`read_history()`, `write_history()`, `add_history()`).
- Custom **GNU Readline** keybindings (`initialize_readline()`):
  - `Up Arrow` (`\e[A` $\to$ `backward_history`): Navigates to older commands while preserving any partially typed current line.
  - `Down Arrow` (`\e[B` $\to$ `forward_history`): Navigates forward through history back to the active command buffer.
  - `Ctrl+A` (`\x01` $\to$ `rl_beg_of_line`) / `Ctrl+E` (`\x05` $\to$ `rl_end_of_line`): Moves the cursor to the start or end of the current command line.

---

#### 4. Malware Detection Command (`sb` — *Squashbug*)
Trojan processes frequently spawn bursts of CPU-intensive child/grandchild processes while the root malware parent sleeps to evade `top`/`htop` inspection.

- **Usage:**
  ```bash
  sb <pid>            # Prints the process ancestry tree (parent, grandparent, ..., init)
  sb <pid> -suggest   # Traverses ancestry and flags the root malware PID using a heuristic score
  ```
- **Heuristic Formula ([`assgn2.squashbug.heuristic.txt`](./assgn2.squashbug.heuristic.txt)):**
  For each candidate process $P$ along the ancestor chain (parsed from `/proc/<pid>/stat` and `/proc/uptime`), the shell computes:

  $$H(P) = \frac{\text{children}(P)}{\text{elapsed minutes}(P)} + \text{CPU}(P) + \sum_{C \in \text{Subtree}_{d \le 5}(P)} \overline{\text{CPU}}(C)$$

  1. **Child Spawning Rate ($\frac{\text{children}(P)}{\text{elapsed minutes}(P)}$):** Captures rapid process proliferation while normalizing by process lifetime (in minutes) so long-running legitimate applications (e.g., browsers) are not falsely penalized.
  2. **Self CPU Utilization ($\text{CPU}(P)$):** Computed from user + kernel clock ticks (`utime + stime` in `/proc/<pid>/stat` divided by `sysconf(_SC_CLK_TCK)`).
  3. **Recursive Descendant CPU Utilization (`find_avg_cpu_of_child`, max depth $5$):** Counters sleeper-parent trojans by recursively aggregating the CPU usage of children and grandchildren up to $5$ generations deep.

- **Testing with [`malware.c`](./malware.c):**
  ```bash
  gcc malware.c -o malware
  ./malware &
  # Inside ./terminal, pass any child/grandchild PID to `sb`:
  sb <child_pid> -suggest
  ```

---

#### 5. Locked File Deletion (`delep` — *Delete with Extreme Prejudice*)
Detects and terminates processes holding open file descriptors or advisory/mandatory `flock` locks on a target file, then deletes the file.

- **Usage:**
  ```bash
  delep <filepath>
  ```
- **Mechanism (`delep()` & `get_process_open_lock_file()` in [`utility.cpp`](./utility.cpp)):**
  1. Forks a dedicated child process connected via an anonymous `pipe()`.
  2. The child scans `/proc/<pid>/fd/*` using `readlink()` to find all PIDs with open descriptors pointing to `<filepath>`, and cross-references `/proc/locks` to identify which of those PIDs hold an active file lock.
  3. The child serializes the PID lists over the pipe back to the parent shell.
  4. The parent displays both lists (processes with the file open and processes locking the file) and prompts `(y/n)`. Upon confirmation (`y`), it sends `SIGKILL` (`kill(pid, SIGKILL)`) to each offending process and unlinks the file via `remove()`.

- **Testing with [`lock.cpp`](./lock.cpp):**
  ```bash
  g++ lock.cpp -o lock
  ./lock hello.txt &
  # Inside ./terminal:
  delep hello.txt
  ```
