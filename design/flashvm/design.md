## System Specification: Framework-Aware Inline Snippet Evaluation Engine## 1. Overview & Strategic Goals
This specification outlines the architecture for a generic, server-side, framework-aware code execution engine designed to support inline code-fixing, debugging, and minor knowledge-check snippet executions directly within a blog, book, or micro-learning platform.
## 1.1 Structural Positioning

* Theia Tier (Existing): Handles heavy, multi-file, long-running project architectures.
* Firecracker Snippet Tier (New): Operates as a fast, stateless "compiler-as-a-service." It runs framework-heavy tasks (e.g., React, Spring Boot) with sub-100ms execution metrics.
* Algorithmic Tier (Existing): Pure logic, scripts, and algorithmic evaluations remain offloaded to the existing Judge0 cluster.

                             [ User Frontend Interface ]
                                          │
                ┌─────────────────────────┼─────────────────────────┐
                ▼                         ▼                         ▼
         [ Theia Tier ]            [ Judge0 Tier ]        [ Firecracker Tier ]
     Long-running workspaces     Algorithmic snippets     Warm Framework Snippets

------------------------------
## 2. Architecture & System Flow
The backend functions as a Warm Stateful Compilation Engine. Instead of booting heavy enterprise frameworks from a cold state, the infrastructure relies on a pre-warmed snapshot lifecycle.
## 2.1 Component Blueprint

   1. TypeScript Backend / Portal: Hosts the web platform, handles authentication, and acts as the gatekeeper for user interaction logs.
   2. Go Controller Microservice: Runs inside a privileged Kubernetes Pod. It converts inbound REST API requests into hardware-virtualized environments via the Firecracker SDK and Linux low-level subsystems.
   3. Firecracker MicroVM (Guest OS): Lightweight microVM instances that contain a custom Linux kernel, warmed-up framework states (Node/Vite or JVM/Spring), and an internal listener daemon.

## 2.2 End-to-End Execution Sequence

[ TypeScript Backend ] ────── POST /api/v1/execute ─────► [ Go Controller Pod ]
                                                               │
     ┌─────────────────────────────────────────────────────────┤
     ▼ (Execution Loop in Goroutine)                           ▼ (Storage Opt)
 1. Create session workspace in RAM disk.                Read from /dev/shm
 2. Clone memory snapshot images.                        (High-speed tmpfs)
 3. Invoke Firecracker CLI via Go SDK.                         │
 4. Load & Resume snapshot state (<30ms).                      │
                                                               ▼
 [ MicroVM Guest OS ] ◄── Stream user code via VSOCK ── [ Active VM Instance ]
         │
         ├──► Guest Agent intercepts snippet payload.
         ├──► Overwrites targeted template class/component file.
         ├──► Framework watcher handles in-memory hot-reload (<50ms).
         ├──► Headless test framework (JUnit/Vitest) runs verification assertions.
         │
         ▼
Return JSON Evaluation Summary Data ──────────────────────────► Pipes back to TS
(VM process reaped & RAM cleared instantly)                     Backend ──► Frontend UI

------------------------------
## 3. Frontend & UX Protocol## 3.1 Controlled Virtual Workspace (No File Creation)

* Users are presented with a single, unified code-editor pane (e.g., Monaco Editor or CodeMirror).
* File tree navigation, folder menus, and structural configuration editing are removed.
* Surrounding file contexts (imports, definitions, annotations) are displayed as greyed-out, read-only code fragments.
* The user's interaction is restricted strictly to an designated Editable Region (e.g., fixing a specific function call, line of logic, or hook configuration).

## 3.2 Live Failure Intercepts

* Snippets load by default into an explicitly broken state.
* Before a reader makes an application change, the interface parses active framework log traces or visual test failures, showing the user exactly what bug needs fixing.

------------------------------
## 4. API Definition## 4.1 Request Payload (TypeScript Backend → Go Controller)

POST /api/v1/execute HTTP/1.1
Host: fc-controller.internal.cluster
Content-Type: application/json

{
  "exercise_id": "spring-di-03",
  "framework": "spring-boot-3",
  "user_snippet": "return new UserServiceImpl(repository);"
}

## 4.2 Response Payload (Go Controller → TypeScript Backend)

{
  "status": "success",
  "exit_code": 0,
  "execution_time_ms": 142,
  "output": "✔ Test passed: UserService successfully injected standard repository instance.",
  "errors": null
}

------------------------------
## 5. Infrastructure & Performance Tuning Rules
To maintain high density and low latency without incurring significant cloud provider infrastructure costs, the Go Controller must follow these requirements:
## 5.1 Host & Compute Prerequisites

* Nested Virtualization: Kubernetes worker nodes must be bare-metal physical configurations or cloud instances explicitly configured with nested virtualization capabilities.
* KVM Mapping: The container execution manifest must explicitly pass through host hardware access to /dev/kvm via local volume mappings and a privileged: true security context.

## 5.2 Micro-Latency Mechanics

* RAM Disk Operations (tmpfs): Base snapshot files (mem.snap, state.snap) must not reside on external or virtual network storage drives (e.g., AWS EBS). They must be copied dynamically into /dev/shm to perform file manipulation loops entirely within host RAM.
* AF_VSOCK Interface Channels: Inter-process data transmission between the host Go Controller and the guest MicroVM must skip the standard Linux network layer. Communication must use internal system sockets (AF_VSOCK) mapped to explicit port protocols.
* Run-and-Die Garbage Collection: MicroVM life-cycles are treated as temporary states. Once evaluation output is recorded from a test runner, the Go Controller forces a termination call (cmd.Process.Kill()), erasing the ephemeral RAM disk workspace immediately.

------------------------------
## 6. Exercise Authoring Format
Creating an interactive, framework-specific debugging snippet exercise requires making an exercise configuration file available to the system cluster. It maps language definitions without project architecture boilerplate:

{
  "exercise_id": "react-useeffect-loop",
  "framework": "react-vite-node20",
  "target_file": "/workspace/src/components/Counter.jsx",
  "marker_string": "// {{USER_CODE_SNIPPET}}",
  "verification_suite": "/workspace/tests/Counter.test.jsx"
}

When a request arrives matching the metadata above, the Go Controller initializes the react-vite-node20 baseline snapshot, swaps the marker_string out with the browser-supplied code payload, and monitors the target verification test suite output inside the guest.
------------------------------
To proceed with implementing this architecture, we should define the infrastructure configuration. Would you like to review the Kubernetes Manifest configuration required to pass through the /dev/kvm device, or should we design the Go Dockerfile runtime wrapper?


