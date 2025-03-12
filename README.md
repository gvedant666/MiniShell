# Process Utilities & Mini Shell — Report-style README

**Short description:**  
A small collection of UNIX utilities and a mini shell implementation for experimenting with process inspection, file locking behavior, and simple process-squashing diagnostics. The project contains test programs that create (or don't create) file locks, a shell with history support and I/O redirection, and helper modules to detect process locks and to locate/diagnose runaway process “bugs”.

---

## Table of contents
- [Overview](#overview)
- [Provided programs / modules](#provided-programs--modules)
- [Key features](#key-features)
- [Build requirements](#build-requirements)
- [Build & quick start](#build--quick-start)
- [Usage examples](#usage-examples)
- [Testing and test utilities](#testing-and-test-utilities)
- [Implementation notes & design decisions](#implementation-notes--design-decisions)

---

## Overview
This repository is a compact, educational codebase demonstrating:
- Creating and observing file-lock behavior (exclusive locks).
- Inspecting `/proc` to enumerate processes and infer locking status.
- A small interactive shell with history and basic job control / I/O redirection parsing.
- A helper (squashbug) that analyzes process trees to locate runaway or suspicious child processes.

The code is written in C++ and plain C-style system calls, intended to be compiled on Linux-like systems where `/proc` and POSIX `flock` are available.

---

## Provided programs / modules
Below are the main source files and their responsibilities:

- `shell.cpp`  
  A mini shell implementation. Implements command parsing, I/O redirection, piping and integrates other utilities from this repo. Uses GNU Readline for interactive history support.

- `history.cpp`, `history.hpp`  
  A small history helper that wraps Readline history functionality and persists commands to `.history` (up to a configured max).

- `delep.cpp`, `delep.hpp`  
  Module that scans `/proc` and inspects process directories to produce a listing of PIDs along with whether they appear to hold a lock (based on presence/read of particular files).

- `squashbug.cpp`, `squashbug.hpp`  
  Implements logic to inspect process trees (children/grandchildren) for suspicious states and help a user identify processes that may need to be terminated. The class is constructed with a target PID and an optional suggestion flag.

- `createlock.cpp`  
  Small utility that opens/creates `lock.txt` and takes an exclusive `flock` (blocking if needed). Useful to test lock-detection behavior.

- `nolock.cpp`  
  Simple program that writes to `lock.txt` but does not hold any `flock` lock. Useful as a contrasting test case.

- `test_squashbug.cpp`  
  Test program that forks a configurable number of children and grandchildren and sleeps — used to create process tree topologies for `squashbug` testing.

---

## Key features
- Demonstrates POSIX `flock` exclusive locking and how locks interact with independent processes.
- `/proc` inspection to collect process metadata (names, states) and heuristics about locks.
- Minimal interactive shell with:
  - Command history persisted to disk.
  - Basic parsing for I/O redirection.
  - Pipeline/foreground execution support (see `shell.cpp` comments).
- Tools intended for testing and demonstration — not a production-grade shell or process manager.

---

## Build requirements
- A modern GNU/Linux toolchain (g++ / gcc).
- `readline` development headers for `history` and `shell` integration.
- Standard C/C++ libraries (`libstdc++`, `libc`).

On Debian/Ubuntu install:
```bash
sudo apt update
sudo apt install build-essential libreadline-dev
```

---

## Build & quick start
There is no provided Makefile, but the programs can be compiled individually. Example commands:

Compile the shell (links with readline):
```bash
g++ -std=c++11 shell.cpp delep.cpp history.cpp -lreadline -o shell
```

Compile squashbug:
```bash
g++ -std=c++11 squashbug.cpp -o squashbug
```

Compile test / smaller utilities:
```bash
g++ -std=c++11 test_squashbug.cpp -o test_squashbug
g++ -std=c++11 createlock.cpp -o createlock
g++ -std=c++11 nolock.cpp -o nolock
g++ -std=c++11 delep.cpp -o delep   # if you want a standalone delep runner
```

If you run into missing headers (for example `ext/stdio_filebuf.h` or other platform-specific headers), adjust compiler flags or comment out small compatibility snippets for your platform.

---

## Usage examples
Open a terminal in the project directory and try the following sequences to explore behavior:

1. Create a process that holds an exclusive lock:
```bash
./createlock
# This will print its PID and then block (holding the lock).
```

2. Run a quick non-locking writer:
```bash
./nolock
# Writes to lock.txt but does not take a flock; useful to contrast.
```

3. From another terminal, run `delep` (or a driver that uses `delep`) to inspect lock state across processes:
```bash
./delep /some/path  # depending on how delep is wired; it writes output to a pipe fd in this build
```

4. Create a noisy process tree (for testing `squashbug`):
```bash
./test_squashbug
# This will fork children/grandchildren and sleep.
# Then run squashbug against one of the PIDs to analyze the tree:
./squashbug <pid>   # see code for expected args (pid, optional suggest flag)
```

5. Run the interactive shell:
```bash
./shell
# Type commands, use arrow keys for history, try simple redirection.
```

---

## Testing
- The included `test_squashbug` and `createlock` / `nolock` programs are intentionally simple stress/test utilities.
- You can write small shell scripts that start `createlock` in background, then run `delep` or `squashbug` against the PID to verify detection behavior.

Example quick test script:
```bash
./createlock &        # holds a lock
CREATOR_PID=$!
sleep 1
./delep /tmp  # or run via the shell to observe reported PID and lock info
kill $CREATOR_PID
```

---

## Implementation notes & design decisions
- The code directly parses `/proc/<pid>/` files to build lightweight process metadata maps. This is portable across Linux flavors, but not across non-Linux OSes.
- `flock` is used in `createlock.cpp` to demonstrate advisory file locks. Since `flock` is advisory, programs must cooperate for it to be meaningful.
- The shell implementation uses `readline` for a better interactive experience and persists a `.history` file (see `history.cpp` / `history.hpp`).
- Error handling in test utilities is intentionally minimal for clarity.

---

## Acknowledgements
- Uses GNU Readline for line editing & history.
- Based on standard POSIX/Linux `/proc` and `flock` facilities.

---
