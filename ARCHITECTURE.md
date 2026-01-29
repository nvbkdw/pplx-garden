# pplx-garden Architecture

This document provides an in-depth overview of the codebase architecture, focusing on the RDMA data transfer implementation and Rust utility libraries.

## Overview

pplx-garden implements a high-performance RDMA (Remote Direct Memory Access) data transfer engine for LLM inference, specifically optimized for Mixture-of-Experts (MoE) dispatch/combine operations. The library supports:

- **AWS EFA** (Elastic Fabric Adapter) via libfabric
- **NVIDIA ConnectX-7** (InfiniBand/RoCE) via libibverbs

## Repository Structure

```
pplx-garden/
├── rust/                           # Utility libraries
│   ├── cuda-lib/                   # CUDA bindings and memory management
│   ├── torch-lib/                  # PyTorch C++ interop via cxx
│   ├── thread-lib/                 # CPU pinning utilities
│   ├── logging-lib/                # Tracing-based logging
│   ├── build-utils/                # Cargo build helpers
│   └── proc-lib/                   # Procedural macros (gpu_test)
├── fabric-lib/                     # RDMA TransferEngine library
│   ├── libfabric-sys/              # FFI bindings to libfabric
│   ├── libibverbs-sys/             # FFI bindings to libibverbs
│   └── fabric-debug/               # Benchmark tool
├── p2p-all-to-all/                 # MoE All-to-All kernels
│   └── a2a-kernels/                # CUDA kernel implementations
├── python-ext/                     # PyO3 Python bindings
└── python/pplx_garden/             # Python package
```

---

## Rust Utility Libraries (`rust/`)

### cuda-lib
CUDA driver and runtime bindings with safe Rust wrappers.

**Key Components:**
- **`cumem.rs`**: Low-level CUDA memory management using `cuMemCreate`/`cuMemMap`
  - `CUMemAllocHandle`: Physical memory allocation with GPUDirect RDMA capability
  - `CUMemMapping`: Virtual address mapping with read/write access
  - `CUMemExport`/`CUMemImport`: Cross-process memory sharing via file descriptors or fabric handles
  - `CUMulticastHandle`: Multi-GPU multicast memory for NVLink-based broadcast

- **`gdr.rs`**: GDRCopy integration for CPU-side polling of GPU memory
  - `GdrCopyContext`: GDRCopy handle management
  - `GdrBuffer`: GPU memory pinned and mapped for CPU access
  - `GdrFlag`: Single-byte flag for GPU→CPU signaling (used for completion notification)
  - `GdrVec<T>`: Typed array for GDRCopy transfers

- **`mem.rs`**: High-level `CudaDeviceMemory` and `CudaHostMemory` wrappers
- **`device.rs`**: `Device` enum (`Host` | `Cuda(CudaDeviceId)`) for device abstraction

### torch-lib
PyTorch C++ interop using cxx-bridge.

**Key Components:**
- `ScalarType`: Rust enum matching PyTorch's scalar types (F16, BF16, F8_E4M3, etc.)
- `from_blob()`: Create PyTorch tensors from raw GPU pointers
- `current_stream()`: Get the current CUDA stream from PyTorch

### thread-lib
Single function `pin_cpu(cpu: usize)` using `pthread_setaffinity_np` for NUMA-aware thread pinning.

### proc-lib
Procedural macro `#[gpu_test]` that:
- Skips tests if no GPU is available
- Appends `_GPU` suffix to function names for test filtering

### build-utils
Cargo build script helpers:
- `find_package()`: Locate library directories via env vars or defaults
- `emit_rerun_if_changed_files()`: Watch source files for incremental builds

---

## fabric-lib Architecture

The RDMA transfer engine is structured in hierarchical layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                      User Application                            │
├─────────────────────────────────────────────────────────────────┤
│              TransferEngine (transfer_engine.rs)                 │
│  • High-level API with callbacks (async support via tokio)       │
│  • submit_transfer(), submit_send(), submit_recv()               │
│  • Manages transfer IDs and completion callbacks                 │
├─────────────────────────────────────────────────────────────────┤
│               FabricEngine (fabric_engine.rs)                    │
│  • Core engine managing multiple WorkerContexts                  │
│  • Routes requests to correct worker based on device             │
│  • Memory region tracking (mr_device_map)                        │
├─────────────────────────────────────────────────────────────────┤
│                 Worker Thread (worker.rs)                        │
│  • One worker per GPU, runs in dedicated pinned thread           │
│  • Receives commands via crossbeam channels                      │
│  • Contains a DomainGroup with N domains                         │
├─────────────────────────────────────────────────────────────────┤
│              DomainGroup<D, N> (domain_group.rs)                 │
│  • Manages N RDMA domains for NIC aggregation                    │
│  • Shards transfers across domains (round-robin or pinned)       │
│  • Handles memory registration across all domains                │
├─────────────────────────────────────────────────────────────────┤
│            RdmaDomain (provider.rs + efa/ or verbs/)             │
│  • Provider-specific implementation                              │
│  • EfaDomain uses libfabric (AWS EFA)                           │
│  • VerbsDomain uses libibverbs (InfiniBand/RoCE)                │
└─────────────────────────────────────────────────────────────────┘
```

### Provider Abstraction

The `RdmaDomain` trait (`provider.rs`) abstracts provider-specific details:

```rust
trait RdmaDomain {
    fn open(info: Self::Info, imm_count_map: Arc<ImmCountMap>) -> Result<Self>;
    fn addr(&self) -> DomainAddress;
    fn register_mr_allow_remote(&mut self, region: &MemoryRegion) -> Result<MemoryRegionRemoteKey>;
    fn submit_write(&mut self, transfer_id: TransferId, dest_addr: DomainAddress, op: WriteOp);
    fn poll_progress(&mut self);
    fn get_completion(&mut self) -> Option<DomainCompletionEntry>;
}
```

**Implementations:**
- `EfaDomain` (`efa/efa_domain.rs`): Uses libfabric APIs (`fi_fabric`, `fi_domain`, `fi_ep`, etc.)
- `VerbsDomain` (`verbs/verbs_domain.rs`): Uses libibverbs APIs (`ibv_create_qp`, etc.)

### Multi-NIC Aggregation

The library supports multiple NICs per GPU via `DomainGroup<D, N>`:

```
GPU 0 ──┬── NIC rdmap1s27 (Domain 0) ───┬──► Remote GPU
        └── NIC rdmap18s28 (Domain 1) ──┘
```

**Routing Options** (`DomainGroupRouting`):
- `RoundRobinSharded { num_shards }`: Split transfer across N NICs
- `Pinned { domain_idx }`: Use a specific NIC

Memory regions are registered on ALL domains in a group, producing a `MemoryRegionDescriptor` with one `(DomainAddress, rkey)` pair per domain.

### Topology Detection (`topo.rs`)

Automatic GPU↔NIC affinity detection:

1. Scan `/sys/bus/pci/devices` for all PCI devices
2. Match GPU and NIC PCI device IDs
3. Find "branching ancestor" (closest common PCI switch)
4. Group NICs with their nearest GPU
5. Assign CPUs from the same NUMA node

```rust
pub struct TopologyGroup {
    pub cuda_device: u8,
    pub numa: u8,
    pub domains: Vec<DomainInfo>,  // NICs for this GPU
    pub cpus: Vec<u16>,            // CPUs for worker pinning
}
```

---

## RDMA Data Flow

### 1. Initialization Flow

```
1. detect_topology()
   └── Scans /sys/bus/pci/devices
   └── Finds GPUs (cudaGetDeviceProperties)
   └── Finds NICs (EFA via libfabric or Verbs via libibverbs)
   └── Matches GPU↔NIC affinity via PCI branching ancestors
   └── Assigns CPUs per NUMA node

2. TransferEngineBuilder
   └── add_gpu_domains(cuda_device, domains, worker_cpu, uvm_cpu)
   └── build() → TransferEngine

3. Worker Spawn
   └── For each GPU: spawn worker thread on pinned CPU
   └── Worker opens RdmaDomains (EFA or Verbs)
   └── Creates: Fabric → Domain → CQ → AV → Endpoint
   └── Returns DomainAddress for remote connections
```

### 2. Memory Registration Flow

```
User: register_memory_allow_remote(ptr, len, Device::Cuda(0))
         │
         ▼
┌─────────────────────┐
│   FabricEngine      │ Routes to correct worker based on device
└─────────────────────┘
         │ WorkerCall::RegisterMRAllowRemote
         ▼
┌─────────────────────┐
│   Worker Thread     │ Receives via crossbeam channel
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│   DomainGroup       │ Registers MR on ALL domains in group
└─────────────────────┘
         │ For each domain:
         ▼
┌─────────────────────┐
│   RdmaDomain        │ Calls fi_mr_regattr (libfabric)
│   (EfaDomain)       │ Returns remote key (rkey)
└─────────────────────┘
         │
         ▼
Returns: (MemoryRegionHandle, MemoryRegionDescriptor)
```

### 3. RDMA Write Transfer Flow

```
User: submit_transfer(TransferRequest::Single(...), callback)
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ TransferEngine                                                   │
│  • Assigns unique TransferId                                     │
│  • Stores callback in DashMap (callbacks.transfer_ops)           │
│  • Forwards to FabricEngine                                      │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ FabricEngine                                                     │
│  • Looks up correct worker by MemoryRegionHandle → Device        │
│  • Sends Box<WorkerCommand::SubmitTransfer> via channel          │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Worker Thread - Main Loop                                        │
│  worker_step():                                                  │
│    1. Process call_rx (function calls)                           │
│    2. Process cmd_rx (transfer commands)                         │
│    3. group.poll_progress() → polls CQs, submits pending ops     │
│    4. group.get_completion() → sends completions to cq_tx        │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ DomainGroup                                                      │
│  submit_single_transfer_request():                               │
│    1. Shard transfer across N domains (round-robin or pinned)    │
│    2. For each shard:                                            │
│       - Create WriteOp::Single with src/dst info                 │
│       - domain.submit_write(transfer_id, dest_addr, WriteOp)     │
│    3. Track in write_ops HashMap for completion aggregation      │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ EfaDomain                                                        │
│  submit_write():                                                 │
│    1. Resolve remote address → fi_addr_t via fi_av_insert        │
│    2. Create WriteOpContext with iterator                        │
│    3. Try eager posting (first op immediately)                   │
│    4. Queue remaining ops if any                                 │
│                                                                  │
│  progress_rdma_write_ops():                                      │
│    Loop: WriteOpContext.peek() → get fi_msg_rma                  │
│          fi_writemsg(ep, msg, flags) → submit to NIC             │
│          WriteOpContext.advance() → next shard/page              │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Hardware: RDMA NIC (EFA/ConnectX)                                │
│  • DMA reads from source GPU memory (GPUDirect RDMA)             │
│  • Transmits over network                                        │
│  • DMA writes to remote GPU memory                               │
│  • Posts completion entry (CQE) locally                          │
│  • If imm_data: posts immediate data to remote CQ                │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Completion Flow

```
Hardware CQ → EfaDomain.poll_cq() → fi_cq_read()
                    │
                    ▼
           handle_cqe(fi_cq_data_entry):
             ├─ FI_WRITE flag → DomainCompletionEntry::Transfer
             ├─ FI_SEND flag  → DomainCompletionEntry::Send
             ├─ FI_RECV flag  → DomainCompletionEntry::Recv
             └─ FI_REMOTE_WRITE → DomainCompletionEntry::ImmData
                    │
                    ▼
           DomainGroup.get_completion():
             • Aggregates completions from all domains
             • When all shards complete → Transfer(id)
                    │
                    ▼
           Worker → cq_tx channel
                    │
                    ▼
           FabricEngine.poll_transfer_completion()
                    │
                    ▼
           TransferEngine callback thread:
             handle_transfer_completion():
               TransferCompletionEntry::Transfer(id)
                 → Remove from callbacks.transfer_ops
                 → Call (callback.on_done)()
```

---

## Transfer Request Types

| Type | Use Case | Operations | GPU-RDMA Path |
|------|----------|------------|---------------|
| **Single** | Large contiguous transfer | 1 RDMA Write | GPU→NIC→Net→NIC→GPU |
| **Paged** | Strided/indexed transfer | N RDMA Writes (per page) | GPU→NIC→Net→NIC→GPU |
| **Scatter** | MoE All-to-All | N Writes to N destinations | 1-to-many |
| **Imm** | Signaling/notification | 0-byte Write with immediate | Minimal data |
| **Barrier** | Multi-target signaling | N 0-byte Writes | 1-to-many |

---

## EFA-Specific Implementation Details

### Domain Initialization (`efa_domain.rs`)

```rust
// Key libfabric objects created:
fabric  = fi_fabric(...)           // Fabric handle
domain  = fi_domain(fabric, ...)   // Domain for MR/CQ/AV
cq      = fi_cq_open(domain, ...)  // Completion queue
av      = fi_av_open(domain, ...)  // Address vector for peer lookup
ep      = fi_endpoint(domain, ...) // RDM endpoint

// Bind CQ and AV to endpoint
fi_ep_bind(ep, cq, FI_SEND | FI_RECV)
fi_ep_bind(ep, av, 0)

// Disable shared memory and CUDA P2P (force RDMA)
fi_setopt(ep, FI_OPT_SHARED_MEMORY_PERMITTED, false)
fi_setopt(ep, FI_OPT_CUDA_API_PERMITTED, false)

// Enable endpoint
fi_control(ep, FI_ENABLE, NULL)
```

### Memory Registration for DMA-BUF

```rust
// For GPU memory with DMA-BUF support (Linux 5.12+)
mr_attr.iface = FI_HMEM_CUDA;
mr_attr.device.cuda = device_id;
mr_attr.dmabuf = &fi_mr_dmabuf { fd: dmabuf_fd, ... };
fi_mr_regattr(domain, &mr_attr, FI_MR_DMABUF, &mr)
```

### Write Operation Posting

```rust
// Eager posting for low latency
WriteOpContext {
    rdma_op_iter: WriteOpIter,  // Iterator over RDMA ops
    cnt_posted_ops: 0,
    cnt_finished_ops: 0,
}

// Post ops with backpressure handling
loop {
    let (msg, flags) = context.rdma_op_iter.peek();
    let ret = fi_writemsg(ep, msg, flags);
    match ret {
        0 => context.rdma_op_iter.advance(),
        FI_EAGAIN => break,  // NIC busy, retry later
    }
}
```

### Immediate Data Counting (`imm_count.rs`)

For synchronization without explicit messages:

```rust
// Sender: RDMA Write with immediate data
WriteOp::Single { imm_data: Some(0x1234), ... }

// Receiver: Count incoming immediates
ImmCountMap::set_expected(imm=0x1234, count=8)  // Expect 8 writes
// When count reached → ImmCountReached(0x1234) completion
```

---

## p2p-all-to-all Architecture

### AllToAllContext (`a2a_context.rs`)

Orchestrates MoE dispatch/combine with RDMA:

```rust
AllToAllContext {
    worker: Arc<WorkerState>,      // RDMA worker thread
    workspace: DeviceWorkspace,    // GPU-side buffers
    transfer_engine: Arc<TransferEngine>,
}

// Four-stage operation:
// 1. dispatch_send() - Route tokens to experts, write to send buffer
// 2. dispatch_recv() - Receive tokens, reorder for local experts
// 3. combine_send()  - Send expert outputs back to sources
// 4. combine_recv()  - Receive and combine expert outputs
```

### CUDA Kernel FFI (`a2a-kernels/src/lib.rs`)

```rust
#[cxx::bridge]
unsafe fn a2a_dispatch_send(...) -> i32;
unsafe fn a2a_dispatch_recv(...) -> i32;
unsafe fn a2a_combine_send(...) -> i32;
unsafe fn a2a_combine_recv(...) -> i32;
```

The kernels:
- Use NVLink for intra-node transfers (via direct GPU memory access)
- Signal RDMA worker via GDRCopy flags for inter-node transfers
- Support CUDA graphs for reduced launch overhead

---

## Key Design Decisions

1. **Static Dispatch for Performance**: `DomainGroup<D, N>` uses const generics to avoid dynamic dispatch in the hot path.

2. **Channel-Based Worker Communication**: Workers receive commands via `crossbeam_channel`, enabling lock-free concurrent submission.

3. **Eager Posting**: First RDMA op is posted immediately on submission; remaining ops queue for progress thread.

4. **Completion Aggregation**: Multi-NIC transfers track per-domain completions; only signal user callback when all shards complete.

5. **Object Pools**: `ObjectPool` for `WriteOpContext` allocation reduces heap churn in the critical path.

6. **Immediate Data for Signaling**: RDMA Write with immediate data enables receiver notification without separate messages.

---

## References

- [RDMA Point-to-Point Communication for LLM Systems](https://arxiv.org/abs/2510.27656) - Research paper
- [DeepEP](https://github.com/deepseek-ai/DeepEP) - MoE kernel inspiration
- [MoonCake](https://www.usenix.org/conference/fast25/presentation/qin) - RDMA library inspiration
