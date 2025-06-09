# Command Processor
- The CP is front-end of the GPU
- Decodes commands from the KMD and dispatches them to the appropriate engines inside the GPU
- Handles HW queue scheduling, command decoding, synchronization, and exception detecting

## Hardware Queue Scheduling
- The CP schedules HW queues
- A HW queue is scheduled only when at least one logical queue is mapped
- Queues tied to different engines are processed by independent schedulers and run in parallel
- HW queues within the same engine share a single scheduler:
    - General queues (graphics, compute) are time-sliced
    - A privileged queue preempts any general queue when its doorbell is written
- After scheduling, the CP processes command packets sequentially from the resident queue
- The CP processes each packet in just a few nanoseconds
    - If it fetched every packet one by one from VRAM, the delay would surface and drag down performance
    - Typical GDDR latency is several hundred nanoseconds
- To hide this latency, the CP keeps an on-chip prefetch buffer that holds a few hundred packets
    - It bulk-fetches packets from VRAM into this buffer and consumes them locally
    - For IB packets, the CP DMA-copies the IB data into the buffer
- When a context switch occurs, any remaining prefetched packets are discarded
    - And the prefetch buffer is refilled from the new queue
    - Although this adds some extra memory traffic, the overall throughput gain is greater

## Job Distribution
- After fetching a packet, the CP moves through decode and dispatch phases before passing the work to the HW pipeline

### Decoding
- Determines the packet type and size
- If the packet is an IB, resolve the IB first

### Dispatching
- Route the decoded work to the appropriate pipeline inside the target engine

## Synchronization
- The CP arbitrates wait/signal operations on sync objects such as fences or semaphores
    - These sync operations are treated as queue operations
- Wait and signal operations follow different processing flows
- For more detailed information on synchronization, refer to the [Synchronization.md](Synchronization.md)

## Exception Detecting
- The CP detects a variety of exceptions and errors that arise during execution
- Detected faults are signaled to the KMD via an IRQ, and the KMD decides how to handle them
- Typical exception types include:
    - Invalid packet: Unknown type, abnormal packet size, or an IB that points to a bad address
    - Protected-memory access: Illegal packet from UMD, or valid packet that references protected memory
    - GPU hang: A queue that remains blocked beyond a timeout threshold
    - Shader trap: Unsupported ISA instructions or arithmetic faults such as divide-by-zero