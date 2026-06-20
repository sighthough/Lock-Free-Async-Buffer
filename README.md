# Lock-Free-Async-Buffer
Technical Specification: Lock-Free 3-Slot Asynchronous Buffer Matrix

# Technical Specification: Lock-Free 3-Slot Asynchronous Buffer Matrix

Try the demo~! 

### Core Objective

To establish a completely lock-free, non-blocking data exchange pipeline between two independent execution threads: a high-frequency **Writer thread** and a variable-frequency **Reader thread**. This architecture enforces a "lossy but fresh" data distribution model, ensuring the Reader always fetches the absolute newest dataset available without causing queue backlogs or forcing the Writer to stall.

---

### Architectural Components

The pipeline utilizes three dedicated, pre-allocated memory slots—**Buffer A, Buffer B, and Buffer C**—managed by three atomic hardware pointers:

* **Writer Pointer (`W_Ptr`):** Tracks the memory slot currently being populated with data.
* **Reader Pointer (`R_Ptr`):** Tracks the memory slot currently being accessed for a read operation.
* **Latest-Ready Pointer (`LR_Ptr`):** Tracks the memory slot containing the absolute newest, fully completed dataset.

---

### The State Rotation Cycle

To eliminate thread collision and data corruption, the system cycles through the three buffers so that the Writer and the Reader never touch the same piece of memory at the same millisecond.

#### Step 1: Initialization (Priming)

Before active looping begins, the Writer fills all three buffers (**A, B, and C**) sequentially. This populates the matrix with valid baseline data so that the Reader can safely sample memory from the very first cycle.

#### Step 2: Active Rotation Execution

Once the pipeline is active, the memory slots cycle through a continuous three-way handshake:

| Writer Action | Reader Action | Pipeline State Logic |
| --- | --- | --- |
| **Writes into Buffer A** | **Reads from Buffer C** | **Buffer B** sits in an idle state, designated as the next available writing block. |
| **Writes into Buffer B** | **Reads from Buffer A** | The Writer completes A, flags it as the newest data, and jumps to B. The Reader finishes C, checks the flags, and claims **Buffer A**. **Buffer C** becomes the idle block. |
| **Writes into Buffer C** | **Reads from Buffer B** | The Writer completes B, flags it as the newest data, and jumps to C. The Reader claims **Buffer B**. **Buffer A** shifts to the idle block. |

This sequence repeats indefinitely. Because there is always a third "idle" buffer waiting in the wings, the Writer can spam high-frequency data updates continuously. If the Reader thread is busy or running slowly, the Writer simply alternates writing between the remaining two buffers, safely overwriting older data and keeping the pipeline perfectly updated.

---

### Thread Synchronization via Atomic Pointer Swaps

To achieve maximum performance and remove the heavy overhead of operating system resource locks (like mutexes), pointer migration is handled entirely via hardware-level **Atomic Pointer Swaps**.

When the Writer thread finishes populating a buffer slot, it does not communicate directly with the Reader. Instead, it executes a single-cycle atomic instruction (such as a CPU-level Compare-And-Swap):

```text
// Step 1: Writer finishes writing data into the current W_Ptr slot.
// Step 2: Atomically swap W_Ptr with LR_Ptr.
// Step 3: Assign W_Ptr to the remaining unshared idle slot index.

```

Because an atomic pointer swap executes in a single hardware clock instruction cycle, it acts as a total **temporal firewall**. The Writer thread never drops an update and never experiences an execution block—while the Reader thread is guaranteed a thread-safe, corruption-free pipeline to the absolute freshest state in memory.



Important Behavior Note: The Telemetry vs. Event Trade-off
Because this 3-slot matrix is designed as a "lossy but fresh" pipeline, it is heavily optimized for continuous streaming data (like mouse coordinate tracking or camera look vectors) where old data loses its value instantly.

However, because the Writer thread freely overwrites older data in the idle slots when the Reader is running slower, any discrete, single-frame interactions (such as mouse clicks, keyboard keystrokes, or UI triggers) can be skipped or missed entirely if they occur during a rapid overwrite cycle.

Why it happens: If the Writer registers a MouseDown event in Buffer A, and then immediately updates the data matrix with a MouseUp event in Buffer B before the Reader thread can execute its next read cycle, the Reader will jump straight to the freshest state (Buffer B) and completely miss the fact that a click ever occurred.

Architectural Fix: For discrete interactions where data loss is unacceptable, this 3-slot buffer should either be paired with a secondary, parallel atomic event queue, or the data packet itself must use a persistent bitmask to accumulate states until the Reader explicitly acknowledges and clears them.


Made by sighthough with the help of googles gemini 3.5 ai
