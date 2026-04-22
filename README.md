# Systems Programming Projects

Three progressively complex projects in C, building from a single-file terminal dashboard to a concurrent, signal-aware multi-process architecture.

---

| Project | Stack | Jump |
|---|---|---|
| Real-Time System Monitor | C · ANSI Escape Codes · `/proc/stat` · `sysinfo(2)` · `utmp` | [↓ Jump](#real-time-system-monitor) |
| File Descriptor Table Inspector | C · `/proc` filesystem · `readlink(2)` · `stat(2)` · `dirent.h` | [↓ Jump](#file-descriptor-table-inspector) |
| Concurrent System Monitor | C · `fork(2)` · `pipe(2)` · `waitpid(2)` · `sigaction(2)` | [↓ Jump](#concurrent-system-monitor) |

---

## Real-Time System Monitor

A terminal dashboard that tracks memory usage, CPU utilization, and connected user sessions; all updating in-place without scrolling.

### What it does
- reads physical RAM via `sysinfo(2)` and converts to GB
- computes CPU utilization by diffing two `/proc/stat` readings across a configurable delay
- lists active user sessions from `/var/run/utmp`
- renders live bar charts using ANSI cursor-positioning so the display refreshes in-place

### CLI
```bash
./monitor [samples [tdelay_us]] [--memory] [--cpu] [--users] [--samples=N] [--tdelay=T]
```

### Key design choices
- positional args (`samples`, `tdelay`) must precede any flags — validated at startup
- if no section flag is given, all three sections display by default
- `#` symbols build up horizontally over time for memory; `:` for CPU — creating a scrolling timeline history within a fixed graph region
- ANSI `\033[row;colH` for cursor movement; `\033[K` to clear to end-of-line — no `system()` calls, no shell commands

## Screenshots
<img width="1512" height="874" alt="Screenshot 2026-02-06 at 6 27 55 PM" src="https://github.com/user-attachments/assets/ae435e40-26be-4f4f-8d57-f76c640f999f" />
<img width="1512" height="878" alt="Screenshot 20<img width="1512 height="866" alt="Screenshot 2026-02-06 at 6 25 14 PM" src="https://github.com/user-attachments/assets/0308fffa-4df3-476f-a6c2-6b362f40c4b2" />
<img width="1512" height="882" alt="Screenshot 2026-02-06 at 6 26 46 PM" src="https://github.com/user-attachments/assets/99ea23e9-cbde-4081-9284-68796569e336" />
<img width="1512" height="866" alt="Screenshot 2026-02-06 at 6 25 14 PM" src="https://github.com/user-attachments/assets/77dab9af-6099-4d55-87dc-61872f90b06c" />

---

## File Descriptor Table Inspector

A Linux tool that reconstructs the OS kernel's file descriptor tables by walking `/proc`, resolving symlinks, and rendering four configurable views.

### What it does
- walks `/proc/<pid>/fd/` for all processes owned by the current user
- resolves each FD symlink with `readlink()` to get its filename/path
- parses inode numbers from symlink targets (e.g. `socket:[12345]`) or falls back to `stat()`
- renders one unified table whose columns are the union of all requested flags
- saves composite table as plain text or raw binary

### CLI
```bash
./fdtable [PID] [--per-process] [--systemWide] [--Vnodes] [--composite] [--summary] [--threshold=X] [--output_TXT] [--output_binary]
```

### Views
| Flag | Columns shown |
|---|---|
| `--per-process` | PID, FD |
| `--systemWide` | PID, FD, Filename |
| `--Vnodes` | FD, Inode |
| `--composite` | Index, PID, FD, Filename, Inode |
| `--summary` | Per-PID FD count |
| `--threshold=X` | PIDs exceeding X open FDs |

### Project structure
```
├── src/
│   ├── main.c <- orchestration only; no logic
│   ├── args.c <- argument parsing
│   ├── fd_table.c <- heap-allocated fixed-size array (max 65,536 entries)
│   ├── fd_collector.c <- /proc walker, inode parser
│   ├── fd_display.c <- all stdout rendering
│   └── fd_output.c <- file saving (txt + binary)
└── include/
    └── *.h
```

### Binary vs. text output: measured comparison
Ran each output mode 7× and averaged `time` results:

| Case | Format | Avg time (s) | File size |
|---|---|---|---|
| All PIDs | `--output_TXT` | 0.0114s | 22,881 B |
| All PIDs | `--output_binary` | 0.0144s | 231,928 B |
| Single PID | `--output_TXT` | 0.0033s | — |
| Single PID | `--output_binary` | 0.0031s | — |

Binary is ~10× larger because each entry is stored at full fixed size (`MAX_PATH_LEN = 1024`) regardless of actual filename length.

## Screenshots
[FD table inspector screenshots.pdf](https://github.com/user-attachments/files/26952554/FD.table.inspector.screenshots.pdf)

---

## Concurrent System Monitor

Extends the first monitor by running each metric collector as a **separate child process**, communicating results back to the parent via pipes, with full signal handling.

### What it does
- for each sample, forks up to 3 children simultaneously — one per enabled section
- each child writes its result to a dedicated pipe, then exits
- parent reads pipes in fixed order (memory → CPU → users), guaranteeing consistent render order regardless of which child finishes first
- CPU worker takes both `/proc/stat` samples internally with `usleep(tdelay)` between them — keeping total runtime within `tdelay × samples` (±1%)
- `SIGTSTP` (Ctrl-Z) is ignored; `SIGINT` (Ctrl-C) prompts the user before quitting

### Concurrency model
```
Per sample:
  fork → worker_memory()   ──┐
  fork → worker_cpu()      ──┤  (run concurrently)
  fork → worker_users()    ──┘
         ↓
  read_all_from_pipe(memory) <- blocks until child exits
  read_all_from_pipe(cpu)
  read_all_from_pipe(users)
         ↓
  waitpid() all children
  render TUI
```

### Pipe protocol
| Worker | Writes |
|---|---|
| `worker_memory` | `"<used_gb> <total_gb>\n"` |
| `worker_cpu` | `"<percent>\n"` |
| `worker_users` | `"username tty (host)\n"` × N, then `"END\n"` |

On any error, workers write sentinel values so the parent never blocks on a read that won't arrive.

### CPU formula
```
Ti = user + nice + sys + idle + iowait + irq + softirq
Ii = idle

CPU% = (1 − ΔI / ΔT) × 100
```
All counters use `long long` (64-bit) to prevent overflow on long-running systems where `/proc/stat` values can exceed `INT_MAX`.

### Project structure
```
├── src/
│   ├── main.c <- fork/pipe orchestration + signal handlers
│   ├── args.c <- argument parsing + validation
│   ├── stats.c <- worker_memory, worker_cpu, worker_users
│   └── display.c <- all ANSI terminal rendering
└── include/
    ├── args.h
    ├── stats.h
    └── display.h
```

### Build
```bash
make # build
make run # build + run with 5 samples at 0.5s
make clean # remove obj/ and binary
make rebuild # clean then build from scratch
```

---

## Progression across projects

| | Monitor v1 | FD Inspector | Monitor v2 |
|---|---|---|---|
| Concurrency | ✗ | ✗ | ✓ fork + pipe per metric |
| Signal handling | ✗ | ✗ | ✓ SIGINT prompt, SIGTSTP ignored |
| Inter-process comms | — | — | ✓ Pipes with sentinel protocol |
| Modular structure | Single file | 6-module split | 4-module split |
| Error handling | Basic | Failures silently skipped | `perror` on every syscall + sentinels |

---

*These were projects I completed for a systems programming course at the University of Toronto Scarborough. Full source code is not posted publicly — if you'd like to discuss the implementation, feel free to reach out.*
