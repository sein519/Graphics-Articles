# Synchronization
- Explain how the sync objects exposed to apps are implemented within the pipeline

## Backend for the API Interface
- A typical API runtime exposes two main sync objects, fence and semaphore
    - Fence: Used for CPU-GPU sync; the GPU signals, the CPU waits
    - Semaphore: Used for sync between GPU queues
- In practice this distinction is conceptual; the back-end implements both with the same logic via a syncobj
- Each syncobj owns a small (32-bit or 64-bit) memory slot that stores its signal state:
    - Fence / binary semaphore: 0 = unsignaled, 1 = signaled
    - Timeline semaphore : monotonically increasing integer value
- The real difference lies in the backing store and the use of host IRQs:
    - Fence: Backing store in host-mapped memory; an IRQ is raised to the host when the GPU signals
    - Semaphore: Backing store in VRAM; the signal is visible only inside the device.
- All sync operations (wait, signal) are queue operations issued through a queue submission

## Wait Operation
- When the KMD enqueues a wait command, the CP processes it
- If SW signaling is required, the KMD also requests that an IRQ be raised when the wait completes
- On encountering the wait command, the CP first checks whether the sync object is already signaled
- If the object is signaled, the wait finishes immediately, so the queue does not block
    - If an IRQ was requested, the CP enqueues the IRQ into the host's IH ring, and the KMD processes it in turn
- If the object is unsignaled, the queue immediately yields its time slice and another queue is scheduled
    - Some implementations perform a short busy-wait before yielding
- The scheduler later grants the blocked queue a new slice
- The CP repeats this procedure until the sync object becomes signaled

## Signal Operation
- When the KMD pushes a signal command into a queue, the CP passes that request to the event generator
- Each queue keeps a per-stage HW counter that tracks the number of in-flight waves
    - Usually each stage has just one counter
    - So a stage flush must be issued to verify that all preceding work has truly finished
    - For this reason, the KMD inserts a stage-partial-flush command at the same time it enqueues the signal command
- As soon as the counter reaches 0, the queue emits a DONE token to the event generator
- Then, the event generator writes a corresponding signal event into its internal event queue
- Finally, the Event DMA consumes that entry and updates the sync object value, completing the signal operation