# Windows Kernel Development Skills

<img width="1600" height="600" alt="windows-kernel-development-skills-hero" src="https://github.com/user-attachments/assets/d394cffc-9257-4bb5-9dd3-9ebd294becc9" />

A collection of focused, reusable **skills** for Windows kernel-mode and device driver development. Each folder covers one topic (IRQL, IRPs, spinlocks, WinDbg, and so on) and can be used as reference material or loaded into an AI coding assistant so it gives accurate, kernel-aware guidance.

> **Warning:** Kernel code runs with full system privileges. A bug can crash the machine or corrupt data. Always develop and test in a virtual machine with a kernel debugger attached, never on your main system.

## Contents

- [What's inside](#whats-inside)
- [Skill index](#skill-index)
- [Getting started](#getting-started)
- [Suggested learning path](#suggested-learning-path)
- [Repository layout](#repository-layout)
- [Contributing](#contributing)
- [License](#license)

## What's inside

The skills cover the full lifecycle of writing a Windows driver:

| Area | What you'll find |
| --- | --- |
| Foundations | OS architecture, environment setup, driver anatomy, build and deploy |
| Core kernel concepts | IRQL, memory pools, strings, lists, exceptions, the Object Manager |
| Concurrency | Spinlocks, synchronization primitives, threads, DPCs, APCs, work items |
| Driver communication | Device objects, IRPs, buffering, IOCTL design, user-mode client protocols |
| Monitoring and callbacks | Process notifications, object callbacks, registry programming |
| Debugging and diagnostics | WinDbg, kernel debugging, crash dump analysis, tracing |

## Skill index

### Foundations

| Skill | Description |
| --- | --- |
| [`wk-windows-architecture`](./wk-windows-architecture) | Windows OS architecture: user mode vs. kernel mode, the executive, and major components |
| [`wk-windows-kernel-programming`](./wk-windows-kernel-programming) | Overview of kernel-mode programming concepts and conventions |
| [`wk-kernel-programming-rules`](./wk-kernel-programming-rules) | Rules and best practices for writing safe, correct kernel code |
| [`wk-driver-dev-environment`](./wk-driver-dev-environment) | Setting up Visual Studio, the WDK, and a test VM |
| [`wk-hello-kernel-driver`](./wk-hello-kernel-driver) | Your first driver: a minimal "Hello, kernel" example |
| [`wk-driver-anatomy`](./wk-driver-anatomy) | Structure of a driver: `DriverEntry`, unload routine, dispatch routines |
| [`wk-driver-build-deploy`](./wk-driver-build-deploy) | Building, signing, installing, and loading drivers |

### Core kernel concepts

| Skill | Description |
| --- | --- |
| [`wk-irql`](./wk-irql) | Interrupt Request Levels and the rules that come with them |
| [`wk-kernel-exceptions`](./wk-kernel-exceptions) | Structured exception handling in kernel mode |
| [`wk-kernel-strings`](./wk-kernel-strings) | `UNICODE_STRING`, `ANSI_STRING`, and safe string handling |
| [`wk-kernel-linked-lists`](./wk-kernel-linked-lists) | `LIST_ENTRY` and the kernel's linked list APIs |
| [`wk-kernel-memory-pools`](./wk-kernel-memory-pools) | Paged and non-paged pool allocation and tagging |
| [`wk-lookaside-lists`](./wk-lookaside-lists) | Lookaside lists for fast fixed-size allocations |
| [`wk-object-manager`](./wk-object-manager) | Kernel objects, handles, reference counting, and object types |

### Concurrency and synchronization

| Skill | Description |
| --- | --- |
| [`wk-spinlocks`](./wk-spinlocks) | Spin locks, queued spin locks, and when to use them |
| [`wk-synchronization`](./wk-synchronization) | Mutexes, events, semaphores, fast mutexes, and executive resources |
| [`wk-kernel-threads`](./wk-kernel-threads) | Creating and managing system threads |
| [`wk-dpcs`](./wk-dpcs) | Deferred Procedure Calls |
| [`wk-apcs`](./wk-apcs) | Asynchronous Procedure Calls |
| [`wk-work-items`](./wk-work-items) | Work items for deferring work to a system thread at `PASSIVE_LEVEL` |

### Driver communication

| Skill | Description |
| --- | --- |
| [`wk-device-objects`](./wk-device-objects) | Creating device objects and symbolic links |
| [`wk-irp-basics`](./wk-irp-basics) | I/O Request Packets, dispatch routines, and completion |
| [`wk-irp-buffering`](./wk-irp-buffering) | Buffered, direct, and neither I/O methods |
| [`wk-ioctl-design`](./wk-ioctl-design) | Designing and validating custom IOCTLs |
| [`wk-driver-client-protocol`](./wk-driver-client-protocol) | Defining the protocol between a driver and its user-mode client |

### Monitoring and callbacks

| Skill | Description |
| --- | --- |
| [`wk-process-notifications`](./wk-process-notifications) | Process and thread creation notifications |
| [`wk-object-callbacks`](./wk-object-callbacks) | Object handle callbacks (`ObRegisterCallbacks`) |
| [`wk-registry-programming`](./wk-registry-programming) | Reading and writing the registry from kernel mode |

### Debugging and diagnostics

| Skill | Description |
| --- | --- |
| [`wk-windbg-fundamentals`](./wk-windbg-fundamentals) | WinDbg basics: commands, symbols, and extensions |
| [`wk-kernel-debugging`](./wk-kernel-debugging) | Live kernel debugging setup and workflow |
| [`wk-crash-dump-analysis`](./wk-crash-dump-analysis) | Analyzing bug checks and crash dumps |
| [`wk-kernel-tracing`](./wk-kernel-tracing) | `DbgPrint`, WPP, and ETW tracing |

## Getting started

Clone the repository:

```bash
git clone https://github.com/XCisCoX/WindowsKernelDevelopmentSkills.git
cd WindowsKernelDevelopmentSkills
```

### As reference material

Browse any `wk-*` folder and read the documentation inside. Each one is self-contained.

### With an AI coding assistant

If your assistant supports skills (for example, Claude Code), copy the skill folders you want into its skills directory:

```bash
# Personal skills, available in all projects
cp -r wk-* ~/.claude/skills/

# Or project-level skills
cp -r wk-* /path/to/your/project/.claude/skills/
```

The assistant can then pull in the relevant skill automatically when you ask about topics like IRQL, IOCTLs, or crash dump analysis.

## Suggested learning path

If you are new to Windows driver development, work through the skills in this order:

1. `wk-windows-architecture`
2. `wk-driver-dev-environment`
3. `wk-hello-kernel-driver`
4. `wk-driver-anatomy`
5. `wk-kernel-programming-rules`
6. `wk-irql`, `wk-kernel-memory-pools`, `wk-kernel-strings`
7. `wk-device-objects`, `wk-irp-basics`, `wk-irp-buffering`, `wk-ioctl-design`
8. `wk-spinlocks`, `wk-synchronization`
9. `wk-windbg-fundamentals`, `wk-kernel-debugging`
10. Everything else, as your project needs it

## Repository layout

```text
WindowsKernelDevelopmentSkills/
├── wk-apcs/
├── wk-crash-dump-analysis/
├── wk-device-objects/
├── wk-dpcs/
├── wk-driver-anatomy/
├── ...
└── wk-work-items/
```

Each `wk-*` directory is one skill, named by topic.

## Prerequisites

To try the examples and techniques covered here, you'll generally need:

- Windows 10 or 11 (64-bit) for development
- [Visual Studio](https://visualstudio.microsoft.com/) with the C++ desktop workload
- The [Windows Driver Kit (WDK)](https://learn.microsoft.com/windows-hardware/drivers/download-the-wdk)
- A virtual machine as the debug target (Hyper-V, VMware, or VirtualBox)
- [WinDbg](https://learn.microsoft.com/windows-hardware/drivers/debugger/)

## Contributing

Contributions are welcome. To add or improve a skill:

1. Fork the repository.
2. Create a branch: `git checkout -b feature/my-change`
3. Keep each skill focused on a single topic and name its folder `wk-<topic>`.
4. Prefer accurate, tested guidance over volume, and cite official Microsoft documentation where possible.
5. Open a pull request describing what you changed and why.

Found an error or an outdated API? Please open an issue.

## Disclaimer

This material is provided for educational purposes, as is, without warranty. You are responsible for testing any driver code before using it anywhere important. Windows and WDK details change between versions, so verify against the [official Microsoft documentation](https://learn.microsoft.com/windows-hardware/drivers/).

## License
License for use is MIT so feel free to use it how ever you want :)
