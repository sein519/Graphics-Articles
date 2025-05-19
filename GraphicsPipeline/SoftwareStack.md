# Software Stack
- Provides a concise overview of the 3D graphics software stack
- Covers layers from modeling to GPU

## Modeling
- Create 3D models using 3rd-party tools such as 3DS Max, Cinema4D, and Blender
- A 3D model typically consists of a meshes and materials
    - Mesh: Represents geometry using data such as vertices, indices, and topologies
    - Material: Represents surface properties such as colors and textures
- Vertices, in particular, include various attributes:
    - Position: Specifies the location of a vertex
    - Normal: Defines the normal vector at a vertex
    - UV: Provides texture mapping coordinates
- Generated 3D models are exported into specific formats, which apps can later import and use
    - Examples include .3ds, .fbx, .obj, and .stl.
- Importing these models into apps typically requires a dedicated import library
    - Third-party import libraries provided by commercial vendors
    - Open-source libraries such as [Assimp](https://github.com/assimp/assimp)

## Application
- Code written by ordinary app developers
- Constructs scenes using content created with modeling tools
- Most bugs that appear at runtime stem from this layer
- App developers build graphics apps by making appropriate calls to the graphics API
- Game engines, including Unity and Unreal Engine, are also app level frameworks

## Compiler
- The compiler usually exists as a module that is separate from the API runtime
    - An app can perform offline compilation with a compiler delivered as a binary (e.g. dxc.exe)
    - An app can also perform online compilation through a compiler supplied as a library (glslang)
- Performs syntax checking on shader code
- Runs optimizations such as loop optimization, dead-code elimination, constant propagation, and branch prediction
- Compiles front-end languages such as GLSL into an IR like SPIR-V
- Some APIs, such as OpenGL, compile shader code at the user-mode driver (UMD)

## API Runtime
- Implementation layer for graphics APIs such as OpenGL, Vulkan, Direct3D, and Metal
- Provides apps with capabilities for computer graphics, including resource creation, state configuration, and issuing draw calls
- Performs state tracking, parameter validation, and error handling, and manages resources visible to app
- Checks shader(IR) for violations of HW constraints

## User-Mode Driver (UMD)
- Backend implementation of the API runtime
- Handles most host-side operations
- The UMD operates entirely in user mode
    - Shares memory space with the app and the API runtime
    - No mode-switch overhead due to lack of elevated privileges
    - A crash in the UMD only affects the corresponding app, not the entire system
    - In contrast, a crash in the KMD may cause system-wide failure
    - Can be replaced at runtime, as it exists simply as a dynamic library
    - Debuggable using common application debugging tools.
- Compiles shader(IR) from the API runtime into GPU-specific ISA
    - Additional low-level optimizations are applied during this process
- Features that are rarely used may not be HW-accelerated by the GPU and thus can be emulated by the UMD
    - The UMD might emulate features like texture border by injecting additional instructions into the user's shader code
    - As some states change, multiple shader permutations may be generated
    - To avoid excessive creation of unused dummy resources, most UMD adopt an on-demand approach
        - Instantiating resources only when actually required
- Manages memory allocation
    - When the API runtime requests memory, the UMD allocates a large memory block from the KMD
    - To minimize allocation overhead, subsequent requests are handled via suballocation from this preallocated memory
    - Explicit APIs like Vulkan and D3D12 may delegate memory management responsibilities to the app
        - However, opaque interfaces such as descriptor pools and command pools remain managed by the UMD
- Creates and submits Indirect Buffers (IB)
    - An IB is a buffer containing a list of commands, preformatted for immediate execution by the CP
    - IB memory blocks allocated by the KMD are suballocated by the UMD, where API-requested commands are recorded
    - IBs typically reside in DMA memory and, once prepared, are submitted to the KMD
        - The KMD passes only the pointer of IB to the CP
        - Before execution, the CP resolves IB contents into VRAM through DMA reads

### Multiple UMD Instances
- A separate UMD instance is loaded for each app that utilizes it
- All these UMD instances must share the GPU as a common resource
- Therefore, the KMD employs time-sharing scheduling among these instances

## Kernel-Mode Driver (KMD)
- Directly controls HW operations
- Unlike the UMD, there is only one instance of the KMD per system
- A crash in the KMD typically results in a system-wide failure
    - In some cases, the system might recover
    - But memory corruption invariably leads to a complete system crash
- Manages unique physical GPU resources:
    - GPU memory allocation
    - Display mode configuration
    - GPU watchdog timer management
    - Interrupt handling
- Responsible for scheduling UMD instances
- Creates command buffers
    - Constructs command buffers directly executable by the CP
    - Unlike IBs, command buffers created by the KMD are submitted directly to the CP
    - Command buffers are recorded by the KMD into a GPU-managed ring buffer
    - it is also called a HW queue
- The ring buffer resides in VRAM and is mapped for KMD access
- When the UMD submits an IB, the KMD places a pointer to the IB into the command buffer for execution by the CP
- The KMD also directly inserts various system commands for GPU into the command buffer
