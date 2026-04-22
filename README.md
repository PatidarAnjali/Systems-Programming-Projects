# Systems Programming Projects

Three progressively complex projects in C, building from a single-file terminal dashboard to a concurrent, signal-aware multi-process architecture.

---

## Table of Contents

- [Overview](#overview)
- [Projects](#projects)
  - [Project 1 – Terminal Dashboard](#project-1--terminal-dashboard)
  - [Project 2 – Multi-Process Monitor](#project-2--multi-process-monitor)
  - [Project 3 – Concurrent Signal-Aware Architecture](#project-3--concurrent-signal-aware-architecture)
- [Prerequisites](#prerequisites)
- [Building & Running](#building--running)
- [License](#license)

---

## Overview

This repository contains three systems programming projects written in C. Each project builds on the previous one, introducing more advanced OS-level concepts such as process management, inter-process communication, signals, and concurrency.

---

## Projects

### Project 1 – Terminal Dashboard

A single-file terminal dashboard that displays live system information (CPU, memory, processes) using standard POSIX APIs.

**Key concepts:** `/proc` filesystem, terminal I/O, ANSI escape codes.

---

### Project 2 – Multi-Process Monitor

Extends the terminal dashboard into a multi-process architecture using `fork()`, `exec()`, and pipes to collect and aggregate system metrics from child processes.

**Key concepts:** `fork`/`exec`, pipes, `waitpid`, inter-process communication.

---

### Project 3 – Concurrent Signal-Aware Architecture

The most advanced project, adding signal handling, concurrent execution, and clean shutdown semantics on top of the multi-process monitor.

**Key concepts:** `sigaction`, signal masks, concurrent processes, graceful termination.

---

## Prerequisites

- GCC (or any C99-compatible compiler)
- GNU Make
- Linux (projects rely on the `/proc` filesystem and POSIX APIs)

```bash
# Debian/Ubuntu
sudo apt-get install build-essential
```

---

## Building & Running

Each project lives in its own subdirectory. Navigate into the project folder and use `make`:

```bash
cd project1/
make
./dashboard
```

```bash
cd project2/
make
./monitor
```

```bash
cd project3/
make
./monitor
```

To clean build artifacts:

```bash
make clean
```

---

## License

This project is provided for educational purposes.
