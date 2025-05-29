# Video Memory
- Explains the unique characteristics of VRAM compared to general-purpose host memory

## Background: GPU VRAM Access Patterns
- Traditionally, GPUs operate on large volumes of data in highly structured ways
    - They continuously read large textures and buffers
    - Memory access patterns tend to be sequential and regular
    - A large number of threads run concurrently
    - Context switching overhead is relatively low
- VRAM access patterns are strongly shaped by this paradigm:
    - I/O bottlenecks can significantly affect performance
    - Burst transfers tend to be long
- Meanwhile, frequent memory exchanges occur between the GPU and the host:
    - Transfer of user parameters (e.g., object transforms, skinning, UI updates)
    - Transfer of textures and vertex/index buffers
    - Transform feedback
    - Offline rendering workloads

## Graphics DDR (GDDR)
- GDDR is a GPU-specific DRAM, and optimized for the characteristics of GPU workloads
- Compared to DDR, GDDR has the following characteristics:
    - High bandwidth
    - High latency

### High Bandwidth
- For GPUs that perform frequent and long-duration I/O operations, higher bandwidth is always beneficial
- GDDR secures high bandwidth by operating many narrow channels, whereas DDR relies on one or two wider channels
    - DDR: 64-bit bus × 1-2 channels = 64-128 bit total
    - GDDR: 32-bit bus × 8–16 channels = 256-512 bit total
- Beyond the raw bandwidth gain, the multi-channel approach provides several advantages:
    - Workloads can be spread across channels, cutting queue latency; stalled channels can be bypassed
    - Narrow buses with large prefetch deliver high pin efficiency
    - Smaller per-channel block sizes simplify modular board designs
        - It allows linear bandwidth scaling with minimal redesign

### High Latency
- GDDR's bandwidth-centric design inevitably brings higher latency
    - Typical causes include:
    - Deeper pipelines to process larger data volumes
    - Larger prefetch units
    - Additional scheduler overhead as channel count increases
- Fortunately, GPUs suffer less from long latency than CPUs because:
    - Context-switch overhead is low, so other threads can run while memory loads are pending
    - Large, deep caches hide many stalls
    - Requests from many threads are merged into single bursts
    - The CP prefetches data before the stage that actually consumes it

## Direct Memory Access (DMA)
- A device maps a portion of system memory and performs R/W operations without host intervention
- Because no CPU cycles are consumed, system load drops and the GPU experiences lower memory latency
- The GPU is one of the most memory-intensive devices, so DMA is critical for performance
- DMA transfers are handled by a dedicated copy engine on the device
- The host allocates pinned memory, passes its address to the device, and the device accesses it over PCIe much like VRAM
- The host can still R/W the same region through VA, enabling zero-copy host-device cooperation