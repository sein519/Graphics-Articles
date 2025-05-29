# Job Scheduling
- Tasks requested by an app pass through a hierarchical scheduling process before GPU executes them

## Engines
- GPUs contain several HW pipelines, each dedicated to a specific set of functions
- These pipelines are collectively called engines, and every engine has its own HW queues and scheduler
- Typical engine types and their responsibilities:
    - Graphics engine: Provides a pipeline for 2D bit-blit operations and 3D  rendering
    - Compute engine: Runs GPGPU kernels
    - Copy engine: Handles memory transfers and pattern fills
        - Including host-device(DMA), device-device, P2P, and zero-fill operations
    - Media engine: Accelerates video encode/decode for formats such as H.264 and AV1
    - Display engine: Manages framebuffer scanning, output timing, and overlay composition

## Hardware Queue
- Each engine owns a fixed number of HW command queues to which work can be submitted
- Strictly speaking, these are set of slots that logical queues can map to and consume
- Commands in a queue are dispatched to the functional blocks inside that engine
- Queues belonging to different engines operate independently
    - A stall in a graphics queue does not halt a compute queue
    - Exceptions arise when explicit dependencies or shared-bandwidth conflicts force serialization
- Queues of the same engine are time-sliced by the dedicated HW scheduler
    - Only one queue is active on that engine at any moment

## Logical Queue
- A logical queue is the queue front end exposed in the UMD
- Each logical queue is bound to a specific engine, and its capabilities depend on that engine type
- The UMD writes the API-requested operations into a command buffer, then submits the buffer to a logical queue
- After submission, the command buffer is encoded into ISA packets and written into an IB located in host-mapped memory
- Every logical queue owns a fixed size ring buffer in VRAM
    - A jump packet inside that ring buffer references the IB
- If # logical queues is less than or equal to # HW queues, each logical queue maps 1:1 to a HW queue
    - No additional scheduling is required
- If there are more logical queues than HW queues, the KMD schedules them

## Benefits of Multiple Queues
- Running several queues inside an engine forces a layered scheduling scheme
- An engine can still execute only one queue at a time, so why accept the extra complexity?
    1. Lower host-side sync overhead
        - If each engine exposed only a single queue, every UMD instance have to contend for that global queue
    2. Higher engine utilization
        - If the current queue is waiting on a memory read, the engine would sit idle
        - Having other ready queues lets the scheduler swap them in, cutting idle time

## Tasks That Bypass the Queues
- CP executes only the commands dispatched through HW queues
- All other operations, such as creating/destroying resources and memory binding, are carried out by the KMD
    - State changes of sync objects (signal and wait) are queue operations and handled by the CP
- Compared with draw or dispatch calls, these management tasks are infrequent and tolerate higher latency
- Light-weight sync (ww-mutexes, spin-locks, atomics, etc.) is sufficient to serialize collisions without impacting performance

## Privileged Queue
- In addition to the command queues exposed to the UMD, there is a KMD-only privileged queue
- It processes system-level requests such as VRAM allocation, power- and clock-state changes, and GPU resets
    - User-mode code can neither see nor control this queue
- Each engine usually has only 1-2 privileged queues
    - Because privileged commands are rare, the design prioritizes responsiveness rather than parallelism
    - Some implementations (like mobile GPUs) use a single global privileged queue
- A privileged queue runs at the highest priority and does not share time slices with regular queues
    - When its doorbell fires, it immediately preempts any regular queue
    - If an app requests VRAM allocation too frequently, regular queues can stall repeatedly, degrading performance