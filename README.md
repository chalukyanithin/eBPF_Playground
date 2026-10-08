# eBPF Playground

### An interactive mental model of how eBPF moves through Linux

**eBPF Playground** is a browser-based interactive visualization for understanding the lifecycle of an eBPF program:

```text
load → verify → JIT → attach → trigger → execute → communicate
```

Instead of presenting eBPF as a collection of disconnected concepts, the playground lets you **follow a program through its lifecycle** and see what happens at each stage.

> This is an educational simulation, not a real eBPF runtime. No programs are loaded into the host kernel and no real kernel events are executed.

---

## Why this exists

eBPF is usually taught through diagrams, documentation, commands, and large code examples.

That creates a problem.

You can understand what the verifier, JIT, hooks, tracepoints, XDP, and ring buffers *are* without having a clear mental model of **how they connect during execution**.

This playground focuses on that missing layer:

**What happened? Where is execution now? What crosses the userspace/kernel boundary? What happens next?**

---

## The interaction model

The primary flow is intentionally simple:

```text
                    USER SPACE
┌───────────────────────────────────────────────────────┐
│                                                       │
│  Application → BPF source/object → Loader / libbpf    │
│                                      │                │
└──────────────────────────────────────┼────────────────┘
                                       │ bpf()
                                       ▼
                    KERNEL SPACE
┌───────────────────────────────────────────────────────┐
│                                                       │
│  Verifier → JIT → Attached Program                    │
│                           │                           │
│                           ▼                           │
│                     Runtime Event                     │
│                           │                           │
│                           ▼                           │
│                    BPF Execution                     │
│                           │                           │
└───────────────────────────┼───────────────────────────┘
                            │
                            ▼
                       Output / Action
```

The UI keeps the current execution state visible instead of hiding it behind multiple menus or abstractions.

The application shows:

- current lifecycle stage
- active execution location
- userspace/kernel boundary
- verifier result
- attachment state
- triggering event
- program execution
- output destination
- execution timeline

---

## Scenarios

### 1. Syscall tracing

The default scenario models a simplified `openat` tracepoint workflow:

```text
Application
    ↓
openat("/etc/hosts")
    ↓
tracepoint/syscalls/sys_enter_openat
    ↓
eBPF program
    ↓
ring buffer
    ↓
userspace consumer
```

The simulated program updates an `open_counts` map and publishes event metadata through a ring buffer.

The project deliberately separates **program loading** from **event execution**:

```text
Program loaded
      ↓
Verifier passed
      ↓
JIT stage
      ↓
Program attached
      ↓
       wait
      ↓
Event triggered
      ↓
BPF executed
      ↓
Ring buffer
      ↓
Userspace consumed
```

---

### 2. Network / XDP

The second scenario models an early packet-processing path:

```text
NIC / RX
   ↓
XDP hook
   ↓
XDP action
   ↓
Network path
```

The playground visualizes actions such as:

- `XDP_PASS`
- `XDP_DROP`
- `XDP_REDIRECT`

Unlike the syscall scenario, the XDP path does **not** require a userspace output step after the action. The simulated packet remains in the kernel execution path.

---

## Safe vs unsafe programs

The playground includes two program variants.

### Safe

A simplified program that passes the modeled verification stage.

### Unsafe

A deliberately invalid example representing a potential memory-safety violation:

```c
SEC("tracepoint/syscalls/sys_enter_openat")
int trace_openat(void *ctx)
{
    // conceptual verifier failure
    char *p = (char *)ctx;
    return p[1024];
}
```

The important interaction is what happens next:

```text
Program
   ↓
Verifier
   ↓
REJECTED
   ✕
JIT
   ✕
Attach
```

The rejected program never reaches attachment or runtime execution.

---

## Step mode

The playground has two execution modes.

### Auto

The lifecycle advances automatically so the entire sequence can be observed as an animated flow.

### Step

The user advances the lifecycle manually:

```text
1. Load
2. Verify
3. JIT
4. Attach
5. Trigger
6. Execute
7. Output / action
```

This is useful when you want to stop at a specific point and understand what the system is doing before continuing.

---

## Inspector

Every major component can be inspected directly.

The inspector exposes information about:

- Application
- BPF source/object
- userspace loader
- verifier
- JIT
- hook
- runtime event
- ring buffer
- NIC / XDP
- XDP program

The goal is not to replace kernel documentation, but to connect the concepts into one coherent execution model.

---

## BPF hooks

The interface includes several common attachment concepts:

```text
Tracepoint
kprobe
uprobe
XDP
TC
cgroup
LSM
```

The visualization also makes an important distinction: different hooks imply different contexts and execution semantics. They are not interchangeable entry points.

---

## Maps and ring buffer

The playground includes a small simulated data view for:

- Hash Map
- Array
- Per-CPU Map
- Ring Buffer

For example:

```text
Hash Map

PID     open() count
4128    17
913      8
7421     3
```

The ring buffer is presented separately as an event-stream mechanism rather than pretending it is just another key/value map.

---

## Design principles

The interface intentionally avoids looking like a generic AI-generated dashboard.

### 1. Execution over decoration

The UI is organized around the actual execution path rather than around cards, metrics, or unnecessary visual effects.

### 2. State should always be visible

At any point, you should be able to answer:

- Is the program loaded?
- Has verification passed?
- Has it reached the JIT stage?
- Is it attached?
- What event triggered it?
- Where is execution currently happening?
- Did data cross into userspace?
- What happens next?

### 3. Userspace and kernel space stay visually separate

The boundary is part of the explanation, not just a visual divider.

### 4. Loading and execution are separate

Attaching an eBPF program does not mean it is continuously executing.

The playground models:

```text
load once
   ↓
attach
   ↓
wait
   ↓
event occurs
   ↓
execute
```

### 5. Simulation is explicit

The events, packet data, PIDs, counters, and execution results shown by the playground are simulated.

---

## Technical implementation

The current implementation is intentionally lightweight:

- HTML
- CSS
- Vanilla JavaScript
- No frontend framework
- No backend
- No external runtime dependency

The application maintains an internal state machine representing:

```text
source
  ↓
bytecode
  ↓
verified
  ↓
JIT
  ↓
attached
  ↓
event triggered
  ↓
executing
  ↓
emitted
  ↓
consumed
```

The UI is then derived from that state.

This makes the visualization deterministic and easy to inspect or extend.

---

## Project structure

The current playground can run as a standalone HTML page.

```text
ebpf-playground/
└── index.html
```

Open `index.html` in a browser.

No build step is required.

---

## What this project is — and isn't

### It is

- an interactive learning tool
- a visual mental model
- a state-machine-driven execution walkthrough
- a way to understand the relationship between userspace, kernel space, hooks, events, and output

### It isn't

- a real eBPF compiler
- a kernel verifier
- a Linux kernel emulator
- a replacement for `bpftool`
- a real packet-processing environment
- a tool that loads programs into your machine's kernel

The source snippets are intentionally simplified and are not complete buildable eBPF programs.

---

## Mental model

If the playground leaves you with one sequence, it should be this:

```text
                LOAD
                  ↓
               VERIFY
                  ↓
                 JIT
                  ↓
                ATTACH
                  ↓
                 WAIT
                  ↓
               TRIGGER
                  ↓
               EXECUTE
                  ↓
          ┌───────┴────────┐
          ↓                ↓
       KERNEL           USERSPACE
       ACTION             OUTPUT
```

That sequence is the core of the playground.

---

## Roadmap

Possible future directions:

- real kernel-backed execution
- actual libbpf examples
- live verifier output
- instruction-level visualization
- BPF bytecode → native JIT visualization
- more hook-specific execution models
- real ring-buffer event streaming
- map mutation visualization
- packet-by-packet XDP execution

These would move the project from an educational model toward an actual interactive eBPF laboratory.

---

## License

Add your preferred license here.