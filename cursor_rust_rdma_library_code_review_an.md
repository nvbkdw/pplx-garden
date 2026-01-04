# Rust RDMA library code review and data flow
_Exported on 1/4/2026 at 13:28:14 PST from Cursor (2.1.46)_

---

**User**

@pplx-garden @fabric-lib 





@learn/pplx-garden/README.md 



this is rust library for RDMA data transferengine, review this codebase and teach me the code structure and teach me RDMA data transfer data flow 

---

**Cursor**

I'll explore the `fabric-lib` codebase to understand its structure and teach you about RDMA data transfer. Let me start by examining the directory structure and key files.

Let me continue exploring the domain group and the EFA/Verbs provider implementations:

Let me examine the EFA domain implementation to understand how RDMA operations are actually performed:

Let me also look at a few more key files to complete the picture:

Let me also look at the debug binary to see usage patterns:

Now I have a comprehensive understanding of the codebase. Let me explain the **code structure** and **RDMA data transfer data flow** for this `fabric-lib` library.

---

# fabric-lib: RDMA TransferEngine Library

## 📁 Code Structure Overview

```
fabric-lib/
├── Cargo.toml              # Library dependencies
├── src/
│   ├── lib.rs              # Public API exports
│   ├── api.rs              # Core API types (TransferRequest, DomainAddress, etc.)
│   ├── interface.rs        # Traits: RdmaEngine, SendRecvEngine, AsyncTransferEngine
│   │
│   ├── transfer_engine.rs          # High-level user-facing engine with callbacks
│   ├── transfer_engine_builder.rs  # Builder pattern for creating TransferEngine
│   ├── fabric_engine.rs            # Core engine managing workers
│   │
│   ├── worker.rs           # Worker threads for RDMA operations
│   ├── domain_group.rs     # Groups multiple domains per GPU for NIC aggregation
│   │
│   ├── provider.rs         # RdmaDomain trait definition
│   ├── provider_dispatch.rs # Dynamic dispatch between EFA/Verbs
│   │
│   ├── efa/                # AWS EFA provider implementation
│   │   ├── mod.rs
│   │   ├── efa_domain.rs   # EFA RDMA domain (libfabric-based)
│   │   ├── efa_devinfo.rs  # EFA device discovery
│   │   ├── efa_mr.rs       # EFA memory region handling
│   │   └── efa_rdma_op.rs  # EFA RDMA operation iterators
│   │
│   ├── verbs/              # InfiniBand/RoCE provider (libibverbs)
│   │   ├── mod.rs
│   │   ├── verbs_domain.rs # Verbs RDMA domain
│   │   ├── verbs_qp.rs     # Queue Pair management
│   │   └── verbs_rdma_op.rs
│   │
│   ├── mr.rs               # Memory Region abstraction (Host/CUDA device)
│   ├── rdma_op.rs          # RDMA operation definitions (WriteOp, SendOp, etc.)
│   ├── topo.rs             # Hardware topology detection (GPU↔NIC affinity)
│   ├── imm_count.rs        # Immediate data counter for synchronization
│   └── utils/              # Helper utilities
│
├── libfabric-sys/          # FFI bindings to libfabric (for EFA)
├── libibverbs-sys/         # FFI bindings to libibverbs (for InfiniBand/RoCE)
└── fabric-debug/           # Benchmark tool using the library
```

---

## 🏛️ Architecture Layers

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

---

## 🔄 RDMA Data Transfer Flow

### 1. Initialization Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    INITIALIZATION FLOW                           │
└─────────────────────────────────────────────────────────────────┘

1. Topology Detection (topo.rs)
   detect_topology()
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
┌─────────────────────────────────────────────────────────────────┐
│                MEMORY REGISTRATION FLOW                          │
└─────────────────────────────────────────────────────────────────┘

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
         │                     │
         │                     └── Contains: ptr + [(addr, rkey), ...]
         │                                   for each domain
         └── Local handle for source buffer
```

### 3. RDMA Write Transfer Flow (Core Data Path)

This is the main data transfer path - **RDMA Write with Immediate**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    RDMA WRITE TRANSFER FLOW                      │
└─────────────────────────────────────────────────────────────────┘

User: submit_transfer(TransferRequest::Single(...), callback)
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ TransferEngine (transfer_engine.rs)                              │
│  • Assigns unique TransferId                                     │
│  • Stores callback in DashMap (callbacks.transfer_ops)           │
│  • Forwards to FabricEngine                                      │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ FabricEngine (fabric_engine.rs)                                  │
│  • Looks up correct worker by MemoryRegionHandle → Device        │
│  • Sends Box<WorkerCommand::SubmitTransfer> via channel          │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Worker Thread - Main Loop (worker.rs)                            │
│  while !stop_signal {                                            │
│      worker_step()  ────┐                                        │
│  }                      │                                        │
│                         │                                        │
│  worker_step():         │                                        │
│    1. Process call_rx (function calls)                           │
│    2. Process cmd_rx (transfer commands) ◄──────────────────┘    │
│    3. group.poll_progress() → polls CQs, submits pending ops     │
│    4. group.get_completion() → sends completions to cq_tx        │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ DomainGroup (domain_group.rs)                                    │
│  submit_single_transfer_request():                               │
│    1. Shard transfer across N domains (round-robin or pinned)    │
│    2. For each shard:                                            │
│       - Create WriteOp::Single with src/dst info                 │
│       - domain.submit_write(transfer_id, dest_addr, WriteOp)     │
│    3. Track in write_ops HashMap for completion aggregation      │
└─────────────────────────────────────────────────────────────────┘
         │ For each shard/domain:
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ EfaDomain (efa/efa_domain.rs)                                    │
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
┌─────────────────────────────────────────────────────────────────┐
│                    COMPLETION FLOW                               │
└─────────────────────────────────────────────────────────────────┘

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

## 🔑 Key Data Types

### Transfer Request Types (api.rs)

```rust
enum TransferRequest {
    Imm(ImmTransferRequest),      // Zero-byte write with immediate data (signaling)
    Single(SingleTransferRequest), // Single contiguous write
    Paged(PagedTransferRequest),   // Scatter-gather with page indices
    Scatter(ScatterTransferRequest), // Multi-destination (All-to-All)
    Barrier(BarrierTransferRequest), // Multi-destination immediate
}
```

### Memory Descriptors

```rust
// Local handle for source memory
struct MemoryRegionHandle { ptr: NonNull<c_void> }

// Remote descriptor shared with peers
struct MemoryRegionDescriptor {
    ptr: u64,
    addr_rkey_list: SmallVec<(DomainAddress, MemoryRegionRemoteKey)>,
    //               └── One entry per domain for NIC aggregation
}
```

---

## 🌐 Multi-NIC Aggregation

The library supports **multiple NICs per GPU** for bandwidth aggregation:

```
GPU 0 ──┬── NIC rdmap1s27 (Domain 0) ───┬──► Remote GPU
        └── NIC rdmap18s28 (Domain 1) ──┘

DomainGroupRouting::RoundRobinSharded { num_shards: 2 }
  → Split transfer across both NICs
  → Each shard uses different domain's (addr, rkey) from descriptor
```

This is handled by `DomainGroup<D, N>` where `N` is a const generic (1, 2, or 4).

---

## 📊 Transfer Types Comparison

| Type | Use Case | Operations | GPU-RDMA Path |
|------|----------|------------|---------------|
| **Single** | Large contiguous transfer | 1 RDMA Write | GPU→NIC→Net→NIC→GPU |
| **Paged** | Strided/indexed transfer | N RDMA Writes (per page) | GPU→NIC→Net→NIC→GPU |
| **Scatter** | MoE All-to-All | N Writes to N destinations | 1-to-many |
| **Imm** | Signaling/notification | 0-byte Write with immediate | Minimal data |

---

## 🔧 Provider Abstraction

The `RdmaDomain` trait abstracts the underlying RDMA provider:

```rust
trait RdmaDomain {
    fn open(info: Self::Info, imm_count_map: Arc<ImmCountMap>) -> Result<Self>;
    fn addr(&self) -> DomainAddress;
    fn register_mr_allow_remote(&mut self, region: &MemoryRegion) -> Result<MemoryRegionRemoteKey>;
    fn submit_write(&mut self, transfer_id: TransferId, dest_addr: DomainAddress, op: WriteOp);
    fn poll_progress(&mut self);
    fn get_completion(&mut self) -> Option<DomainCompletionEntry>;
    // ...
}
```

- **EfaDomain**: Uses `libfabric` for AWS EFA
- **VerbsDomain**: Uses `libibverbs` for InfiniBand/RoCE (ConnectX)

The worker thread dynamically selects the provider based on discovered devices.

---

This library is optimized for **LLM inference** where low-latency GPU-to-GPU communication is critical, especially for MoE (Mixture-of-Experts) dispatch/combine operations that require scatter/gather patterns across many GPUs.

```plaintext
fabric-lib/
├── Cargo.toml              # Library dependencies
├── src/
│   ├── lib.rs              # Public API exports
│   ├── api.rs              # Core API types (TransferRequest, DomainAddress, etc.)
│   ├── interface.rs        # Traits: RdmaEngine, SendRecvEngine, AsyncTransferEngine
│   │
│   ├── transfer_engine.rs          # High-level user-facing engine with callbacks
│   ├── transfer_engine_builder.rs  # Builder pattern for creating TransferEngine
│   ├── fabric_engine.rs            # Core engine managing workers
│   │
│   ├── worker.rs           # Worker threads for RDMA operations
│   ├── domain_group.rs     # Groups multiple domains per GPU for NIC aggregation
│   │
│   ├── provider.rs         # RdmaDomain trait definition
│   ├── provider_dispatch.rs # Dynamic dispatch between EFA/Verbs
│   │
│   ├── efa/                # AWS EFA provider implementation
│   │   ├── mod.rs
│   │   ├── efa_domain.rs   # EFA RDMA domain (libfabric-based)
│   │   ├── efa_devinfo.rs  # EFA device discovery
│   │   ├── efa_mr.rs       # EFA memory region handling
│   │   └── efa_rdma_op.rs  # EFA RDMA operation iterators
│   │
│   ├── verbs/              # InfiniBand/RoCE provider (libibverbs)
│   │   ├── mod.rs
│   │   ├── verbs_domain.rs # Verbs RDMA domain
│   │   ├── verbs_qp.rs     # Queue Pair management
│   │   └── verbs_rdma_op.rs
│   │
│   ├── mr.rs               # Memory Region abstraction (Host/CUDA device)
│   ├── rdma_op.rs          # RDMA operation definitions (WriteOp, SendOp, etc.)
│   ├── topo.rs             # Hardware topology detection (GPU↔NIC affinity)
│   ├── imm_count.rs        # Immediate data counter for synchronization
│   └── utils/              # Helper utilities
│
├── libfabric-sys/          # FFI bindings to libfabric (for EFA)
├── libibverbs-sys/         # FFI bindings to libibverbs (for InfiniBand/RoCE)
└── fabric-debug/           # Benchmark tool using the library
```

```plaintext
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

```plaintext
┌─────────────────────────────────────────────────────────────────┐
│                    INITIALIZATION FLOW                           │
└─────────────────────────────────────────────────────────────────┘

1. Topology Detection (topo.rs)
   detect_topology()
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

```plaintext
┌─────────────────────────────────────────────────────────────────┐
│                MEMORY REGISTRATION FLOW                          │
└─────────────────────────────────────────────────────────────────┘

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
         │                     │
         │                     └── Contains: ptr + [(addr, rkey), ...]
         │                                   for each domain
         └── Local handle for source buffer
```

```plaintext
┌─────────────────────────────────────────────────────────────────┐
│                    RDMA WRITE TRANSFER FLOW                      │
└─────────────────────────────────────────────────────────────────┘

User: submit_transfer(TransferRequest::Single(...), callback)
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ TransferEngine (transfer_engine.rs)                              │
│  • Assigns unique TransferId                                     │
│  • Stores callback in DashMap (callbacks.transfer_ops)           │
│  • Forwards to FabricEngine                                      │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ FabricEngine (fabric_engine.rs)                                  │
│  • Looks up correct worker by MemoryRegionHandle → Device        │
│  • Sends Box<WorkerCommand::SubmitTransfer> via channel          │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Worker Thread - Main Loop (worker.rs)                            │
│  while !stop_signal {                                            │
│      worker_step()  ────┐                                        │
│  }                      │                                        │
│                         │                                        │
│  worker_step():         │                                        │
│    1. Process call_rx (function calls)                           │
│    2. Process cmd_rx (transfer commands) ◄──────────────────┘    │
│    3. group.poll_progress() → polls CQs, submits pending ops     │
│    4. group.get_completion() → sends completions to cq_tx        │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ DomainGroup (domain_group.rs)                                    │
│  submit_single_transfer_request():                               │
│    1. Shard transfer across N domains (round-robin or pinned)    │
│    2. For each shard:                                            │
│       - Create WriteOp::Single with src/dst info                 │
│       - domain.submit_write(transfer_id, dest_addr, WriteOp)     │
│    3. Track in write_ops HashMap for completion aggregation      │
└─────────────────────────────────────────────────────────────────┘
         │ For each shard/domain:
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ EfaDomain (efa/efa_domain.rs)                                    │
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

```plaintext
┌─────────────────────────────────────────────────────────────────┐
│                    COMPLETION FLOW                               │
└─────────────────────────────────────────────────────────────────┘

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

```rust
enum TransferRequest {
    Imm(ImmTransferRequest),      // Zero-byte write with immediate data (signaling)
    Single(SingleTransferRequest), // Single contiguous write
    Paged(PagedTransferRequest),   // Scatter-gather with page indices
    Scatter(ScatterTransferRequest), // Multi-destination (All-to-All)
    Barrier(BarrierTransferRequest), // Multi-destination immediate
}
```

```rust
// Local handle for source memory
struct MemoryRegionHandle { ptr: NonNull<c_void> }

// Remote descriptor shared with peers
struct MemoryRegionDescriptor {
    ptr: u64,
    addr_rkey_list: SmallVec<(DomainAddress, MemoryRegionRemoteKey)>,
    //               └── One entry per domain for NIC aggregation
}
```

```plaintext
GPU 0 ──┬── NIC rdmap1s27 (Domain 0) ───┬──► Remote GPU
        └── NIC rdmap18s28 (Domain 1) ──┘

DomainGroupRouting::RoundRobinSharded { num_shards: 2 }
  → Split transfer across both NICs
  → Each shard uses different domain's (addr, rkey) from descriptor
```

```rust
trait RdmaDomain {
    fn open(info: Self::Info, imm_count_map: Arc<ImmCountMap>) -> Result<Self>;
    fn addr(&self) -> DomainAddress;
    fn register_mr_allow_remote(&mut self, region: &MemoryRegion) -> Result<MemoryRegionRemoteKey>;
    fn submit_write(&mut self, transfer_id: TransferId, dest_addr: DomainAddress, op: WriteOp);
    fn poll_progress(&mut self);
    fn get_completion(&mut self) -> Option<DomainCompletionEntry>;
    // ...
}
```

---

**User**

I want to implement a distributed Key value cache, with a master node recording the metadata of key and value location, and worker nodes store values as byte block in memory, when one node read a key, it can either fetch from local memory or fetch from remote nodes using this RMDA transfer engine, teach me how to implement this RDMA transfer between nodes and how to integrate it in this distributed key value cache

---

**Cursor**

# Distributed Key-Value Cache with RDMA Transfer

I'll teach you how to implement this distributed KV cache step by step. Let me design the architecture and show you how to integrate the `fabric-lib` RDMA transfer engine.

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           MASTER NODE                                     │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                     Metadata Store                                  │  │
│  │  HashMap<Key, ValueLocation>                                        │  │
│  │                                                                     │  │
│  │  ValueLocation {                                                    │  │
│  │      node_id: u32,                                                  │  │
│  │      mr_descriptor: MemoryRegionDescriptor,  // For RDMA access    │  │
│  │      offset: u64,                                                   │  │
│  │      length: u64,                                                   │  │
│  │  }                                                                  │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                    Control Plane (gRPC/TCP)
                    ┌───────────────┴───────────────┐
                    │               │               │
                    ▼               ▼               ▼
┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐
│     WORKER NODE 0    │ │     WORKER NODE 1    │ │     WORKER NODE 2    │
│                      │ │                      │ │                      │
│  ┌────────────────┐  │ │  ┌────────────────┐  │ │  ┌────────────────┐  │
│  │ Value Storage  │  │ │  │ Value Storage  │  │ │  │ Value Storage  │  │
│  │ (GPU/CPU Mem)  │  │ │  │ (GPU/CPU Mem)  │  │ │  │ (GPU/CPU Mem)  │  │
│  │                │  │ │  │                │  │ │  │                │  │
│  │ RDMA-Registered│  │ │  │ RDMA-Registered│  │ │  │ RDMA-Registered│  │
│  └────────────────┘  │ │  └────────────────┘  │ │  └────────────────┘  │
│          │           │ │          │           │ │          │           │
│  ┌────────────────┐  │ │  ┌────────────────┐  │ │  ┌────────────────┐  │
│  │ TransferEngine │  │ │  │ TransferEngine │  │ │  │ TransferEngine │  │
│  └────────────────┘  │ │  └────────────────┘  │ │  └────────────────┘  │
│          │           │ │          │           │ │          │           │
│      RDMA NIC        │ │      RDMA NIC        │ │      RDMA NIC        │
└──────────┬───────────┘ └──────────┬───────────┘ └──────────┬───────────┘
           │                        │                        │
           └────────── RDMA Network (Data Plane) ────────────┘
```

---

## 📝 Step-by-Step Implementation

### Step 1: Define Core Data Structures

```rust
// src/types.rs
use bytes::Bytes;
use fabric_lib::api::{DomainAddress, MemoryRegionDescriptor};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::Arc;
use parking_lot::RwLock;

/// Unique identifier for each worker node
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct NodeId(pub u32);

/// Key type for the cache (could be a hash or structured key)
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct CacheKey(pub Bytes);

/// Location of a value in the distributed cache
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ValueLocation {
    /// Which node stores this value
    pub node_id: NodeId,
    /// RDMA memory region descriptor for remote access
    pub mr_descriptor: MemoryRegionDescriptor,
    /// Offset within the memory region
    pub offset: u64,
    /// Length of the value in bytes
    pub length: u64,
}

/// Metadata about a node's RDMA endpoint
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NodeInfo {
    pub node_id: NodeId,
    /// RDMA address for this node (one per domain/NIC)
    pub rdma_addresses: Vec<DomainAddress>,
    /// Number of NICs per GPU
    pub nets_per_gpu: u8,
}

/// Master's view of the cluster
pub struct ClusterMetadata {
    /// Key to location mapping
    pub key_locations: RwLock<HashMap<CacheKey, ValueLocation>>,
    /// Node info for RDMA connections
    pub nodes: RwLock<HashMap<NodeId, NodeInfo>>,
}
```

### Step 2: Worker Node - RDMA Setup and Memory Management

```rust
// src/worker/mod.rs
use anyhow::Result;
use fabric_lib::{
    TransferEngine, TransferEngineBuilder, TransferCallback,
    detect_topology,
    api::{
        DomainAddress, MemoryRegionHandle, MemoryRegionDescriptor,
        SingleTransferRequest, TransferRequest, DomainGroupRouting,
        PeerGroupHandle, SmallVec,
    },
};
use cuda_lib::{Device, CudaDeviceMemory, CudaDeviceId};
use std::collections::HashMap;
use std::ffi::c_void;
use std::num::NonZeroU8;
use std::ptr::NonNull;
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, AtomicI64, Ordering};
use parking_lot::RwLock;

/// Configuration for the cache worker
pub struct WorkerConfig {
    pub node_id: NodeId,
    pub gpu_ids: Vec<u8>,
    pub nets_per_gpu: usize,
    pub block_size: usize,      // Size of each cache block
    pub num_blocks: usize,      // Number of blocks per GPU
}

/// A block of memory that can store a cache value
pub struct CacheBlock {
    pub offset: u64,
    pub length: u64,
    pub in_use: bool,
}

/// Memory pool for a single GPU
pub struct GpuMemoryPool {
    pub gpu_id: u8,
    pub memory: CudaDeviceMemory,
    pub mr_handle: MemoryRegionHandle,
    pub mr_descriptor: MemoryRegionDescriptor,
    pub blocks: Vec<CacheBlock>,
    pub free_blocks: Vec<usize>,
}

/// The RDMA-enabled cache worker
pub struct CacheWorker {
    pub node_id: NodeId,
    pub config: WorkerConfig,
    pub engine: Arc<TransferEngine>,
    pub memory_pools: Vec<GpuMemoryPool>,
    /// Mapping of remote nodes to their peer group handles
    pub peer_groups: RwLock<HashMap<NodeId, PeerGroupHandle>>,
    /// Local key-value index
    pub local_index: RwLock<HashMap<CacheKey, (usize, usize)>>, // (gpu_idx, block_idx)
}

impl CacheWorker {
    /// Initialize the RDMA cache worker
    pub fn new(config: WorkerConfig) -> Result<Self> {
        // Step 1: Detect hardware topology
        let topo_groups = detect_topology()?;
        
        // Step 2: Build TransferEngine with selected GPUs
        let mut builder = TransferEngineBuilder::default();
        for &gpu_id in &config.gpu_ids {
            let topo = topo_groups
                .iter()
                .find(|t| t.cuda_device == gpu_id)
                .ok_or_else(|| anyhow::anyhow!("GPU {} not found", gpu_id))?;
            
            let domains: Vec<_> = topo.domains
                .iter()
                .take(config.nets_per_gpu)
                .cloned()
                .collect();
            
            let worker_cpu = topo.cpus[0];
            let uvm_cpu = topo.cpus[1];
            
            builder.add_gpu_domains(gpu_id, domains, worker_cpu, uvm_cpu);
        }
        let engine = Arc::new(builder.build()?);
        
        // Step 3: Allocate and register memory pools
        let mut memory_pools = Vec::new();
        for &gpu_id in &config.gpu_ids {
            let pool = Self::create_memory_pool(
                &engine,
                gpu_id,
                config.block_size,
                config.num_blocks,
            )?;
            memory_pools.push(pool);
        }
        
        Ok(Self {
            node_id: config.node_id,
            config,
            engine,
            memory_pools,
            peer_groups: RwLock::new(HashMap::new()),
            local_index: RwLock::new(HashMap::new()),
        })
    }
    
    /// Create a memory pool for a GPU
    fn create_memory_pool(
        engine: &TransferEngine,
        gpu_id: u8,
        block_size: usize,
        num_blocks: usize,
    ) -> Result<GpuMemoryPool> {
        use cuda_lib::rt::cudaSetDevice;
        
        // Set CUDA device
        cudaSetDevice(gpu_id as i32)?;
        
        // Allocate GPU memory
        let total_size = block_size * num_blocks;
        let memory = CudaDeviceMemory::device(total_size)?;
        
        // Register with RDMA engine (allow remote access)
        let (mr_handle, mr_descriptor) = engine.register_memory_allow_remote(
            memory.ptr(),
            total_size,
            Device::Cuda(CudaDeviceId(gpu_id)),
        )?;
        
        // Initialize block tracking
        let blocks: Vec<_> = (0..num_blocks)
            .map(|i| CacheBlock {
                offset: (i * block_size) as u64,
                length: block_size as u64,
                in_use: false,
            })
            .collect();
        let free_blocks: Vec<_> = (0..num_blocks).collect();
        
        Ok(GpuMemoryPool {
            gpu_id,
            memory,
            mr_handle,
            mr_descriptor,
            blocks,
            free_blocks,
        })
    }
    
    /// Get node info for registration with master
    pub fn get_node_info(&self) -> NodeInfo {
        NodeInfo {
            node_id: self.node_id,
            rdma_addresses: self.get_all_domain_addresses(),
            nets_per_gpu: self.engine.nets_per_gpu().get(),
        }
    }
    
    /// Get RDMA addresses for all domains
    fn get_all_domain_addresses(&self) -> Vec<DomainAddress> {
        // The main address represents this node
        // In practice, you'd collect addresses from all domains
        vec![self.engine.main_address()]
    }
    
    /// Allocate a block for storing a value
    pub fn allocate_block(&self, gpu_idx: usize) -> Option<(usize, ValueLocation)> {
        let pool = &self.memory_pools[gpu_idx];
        
        // Find a free block (simplified - real impl needs proper locking)
        let block_idx = pool.free_blocks.first().copied()?;
        let block = &pool.blocks[block_idx];
        
        let location = ValueLocation {
            node_id: self.node_id,
            mr_descriptor: pool.mr_descriptor.clone(),
            offset: block.offset,
            length: block.length,
        };
        
        Some((block_idx, location))
    }
    
    /// Register a remote node for RDMA access
    pub fn register_remote_node(&self, remote_info: &NodeInfo) -> Result<()> {
        // Build address list for peer group
        // Each entry in SmallVec corresponds to one local domain
        let addrs: Vec<SmallVec<DomainAddress>> = remote_info.rdma_addresses
            .iter()
            .map(|addr| {
                let mut sv = SmallVec::new();
                sv.push(addr.clone());
                sv
            })
            .collect();
        
        // Add peer group to engine
        let handle = self.engine.add_peer_group(
            addrs,
            Device::Cuda(CudaDeviceId(self.config.gpu_ids[0])),
        )?;
        
        self.peer_groups.write().insert(remote_info.node_id, handle);
        Ok(())
    }
}
```

### Step 3: RDMA Read from Remote Node

This is the **key operation** - fetching a value from a remote node using RDMA:

```rust
// src/worker/rdma_ops.rs
impl CacheWorker {
    /// Fetch a value from a remote node using RDMA
    /// This is a ONE-SIDED operation - no involvement from remote CPU!
    pub async fn fetch_remote_value(
        &self,
        remote_location: &ValueLocation,
        local_gpu_idx: usize,
        local_block_idx: usize,
    ) -> Result<()> {
        let local_pool = &self.memory_pools[local_gpu_idx];
        let local_block = &local_pool.blocks[local_block_idx];
        
        // Build the transfer request
        // Note: fabric-lib uses RDMA WRITE (push model)
        // For a "pull" model (remote read), we need coordination
        
        // Option 1: Have the remote node push data to us
        // Option 2: Use Send/Recv for coordination + Write for data
        
        // Here's the RDMA WRITE approach (remote node pushes to us):
        let request = TransferRequest::Single(SingleTransferRequest {
            // Source: remote node's memory
            src_mr: MemoryRegionHandle::new(
                NonNull::new(remote_location.mr_descriptor.ptr as *mut c_void).unwrap()
            ),
            src_offset: remote_location.offset,
            length: remote_location.length,
            imm_data: None,
            // Destination: our local memory
            dst_mr: local_pool.mr_descriptor.clone(),
            dst_offset: local_block.offset,
            domain: DomainGroupRouting::RoundRobinSharded {
                num_shards: self.engine.nets_per_gpu(),
            },
        });
        
        // Submit and wait for completion
        self.engine.submit_transfer_async(request).await?;
        
        Ok(())
    }
    
    /// Push a local value to a remote node
    pub fn push_to_remote(
        &self,
        local_gpu_idx: usize,
        local_block_idx: usize,
        remote_location: &ValueLocation,
    ) -> Result<tokio::sync::oneshot::Receiver<Result<()>>> {
        let local_pool = &self.memory_pools[local_gpu_idx];
        let local_block = &local_pool.blocks[local_block_idx];
        
        // Create completion channel
        let (tx, rx) = tokio::sync::oneshot::channel();
        
        // Build RDMA WRITE request
        let request = TransferRequest::Single(SingleTransferRequest {
            src_mr: local_pool.mr_handle,
            src_offset: local_block.offset,
            length: local_block.length.min(remote_location.length),
            imm_data: Some(0x1), // Signal completion to remote
            dst_mr: remote_location.mr_descriptor.clone(),
            dst_offset: remote_location.offset,
            domain: DomainGroupRouting::RoundRobinSharded {
                num_shards: self.engine.nets_per_gpu(),
            },
        });
        
        // Submit with callback
        let callback = TransferCallback {
            on_done: Box::new(move || {
                let _ = tx.send(Ok(()));
                Ok(())
            }),
            on_error: Box::new(move |e| {
                // tx already moved, can't send error
                Err(format!("RDMA transfer failed: {:?}", e))
            }),
        };
        
        self.engine.submit_transfer(request, callback)?;
        
        Ok(rx)
    }
    
    /// Fetch with atomic counter for batch operations
    pub fn fetch_batch_atomic(
        &self,
        requests: Vec<(ValueLocation, usize, usize)>, // (remote, local_gpu, local_block)
        completion_counter: Arc<AtomicI64>,
        error_counter: Arc<AtomicI64>,
    ) -> Result<()> {
        for (remote_location, local_gpu_idx, local_block_idx) in requests {
            let local_pool = &self.memory_pools[local_gpu_idx];
            let local_block = &local_pool.blocks[local_block_idx];
            
            let request = TransferRequest::Single(SingleTransferRequest {
                src_mr: local_pool.mr_handle,
                src_offset: local_block.offset,
                length: remote_location.length,
                imm_data: None,
                dst_mr: remote_location.mr_descriptor.clone(),
                dst_offset: remote_location.offset,
                domain: DomainGroupRouting::RoundRobinSharded {
                    num_shards: self.engine.nets_per_gpu(),
                },
            });
            
            self.engine.submit_transfer_atomic(
                request,
                completion_counter.clone(),
                error_counter.clone(),
            )?;
        }
        
        Ok(())
    }
    
    /// Wait for batch completion using spin-wait
    pub fn wait_batch_completion(
        completion_counter: &AtomicI64,
        expected: i64,
    ) {
        while completion_counter.load(Ordering::Acquire) < expected {
            std::hint::spin_loop();
        }
    }
}
```

### Step 4: Request-Response Protocol for Remote Reads

Since RDMA Write is a "push" model, we need a protocol for "pull" (reading from remote):

```rust
// src/protocol.rs
use serde::{Deserialize, Serialize};

/// Messages sent between nodes
#[derive(Serialize, Deserialize)]
pub enum CacheMessage {
    /// Request to read a value - receiver should RDMA write back
    ReadRequest {
        request_id: u64,
        key: CacheKey,
        /// Where to write the response (requestor's memory)
        response_location: ValueLocation,
    },
    
    /// Acknowledgment that write is complete
    ReadResponse {
        request_id: u64,
        success: bool,
        error: Option<String>,
    },
    
    /// Request to write a value
    WriteRequest {
        request_id: u64,
        key: CacheKey,
        /// Where the value is (sender's memory) - receiver will RDMA read
        /// OR sender can push directly
        value_location: ValueLocation,
    },
    
    /// Node registration
    Register(NodeInfo),
}

// src/worker/protocol.rs
impl CacheWorker {
    /// Handle incoming message (via Send/Recv)
    pub async fn handle_message(&self, msg: CacheMessage) -> Result<Option<CacheMessage>> {
        match msg {
            CacheMessage::ReadRequest { request_id, key, response_location } => {
                // Look up local value
                let local_index = self.local_index.read();
                if let Some(&(gpu_idx, block_idx)) = local_index.get(&key) {
                    drop(local_index);
                    
                    // RDMA WRITE the value to requestor's memory
                    let rx = self.push_to_remote(gpu_idx, block_idx, &response_location)?;
                    rx.await??;
                    
                    return Ok(Some(CacheMessage::ReadResponse {
                        request_id,
                        success: true,
                        error: None,
                    }));
                }
                
                Ok(Some(CacheMessage::ReadResponse {
                    request_id,
                    success: false,
                    error: Some("Key not found".to_string()),
                }))
            }
            
            CacheMessage::WriteRequest { request_id, key, value_location } => {
                // Allocate local block
                if let Some((block_idx, local_location)) = self.allocate_block(0) {
                    // Could either:
                    // 1. Have sender push to us (preferred for RDMA)
                    // 2. We fetch from sender
                    
                    // Store in local index
                    self.local_index.write().insert(key, (0, block_idx));
                }
                Ok(None)
            }
            
            _ => Ok(None),
        }
    }
    
    /// Request a value from a remote node (full protocol)
    pub async fn request_remote_value(
        &self,
        remote_node_id: NodeId,
        remote_node_addr: DomainAddress,
        key: CacheKey,
    ) -> Result<Vec<u8>> {
        // 1. Allocate local buffer for response
        let (block_idx, response_location) = self.allocate_block(0)
            .ok_or_else(|| anyhow::anyhow!("No free blocks"))?;
        
        // 2. Send read request (using Send/Recv messaging)
        let request = CacheMessage::ReadRequest {
            request_id: rand::random(),
            key,
            response_location,
        };
        
        // Serialize and send via RDMA Send
        let data = postcard::to_vec(&request)?;
        self.send_message(remote_node_addr.clone(), &data).await?;
        
        // 3. Wait for response message
        let response_data = self.recv_message().await?;
        let response: CacheMessage = postcard::from_bytes(&response_data)?;
        
        // 4. Read data from local buffer
        let pool = &self.memory_pools[0];
        let block = &pool.blocks[block_idx];
        
        // Copy from GPU to host
        let mut result = vec![0u8; block.length as usize];
        cuda_memcpy_d2h(&mut result, pool.memory.ptr(), block.offset as usize)?;
        
        Ok(result)
    }
    
    /// Low-level send message
    async fn send_message(&self, dest: DomainAddress, data: &[u8]) -> Result<()> {
        // Need a registered host buffer for Send/Recv
        // This would be allocated during worker init
        todo!("Implement using engine.submit_send_async()")
    }
    
    /// Low-level receive message
    async fn recv_message(&self) -> Result<Vec<u8>> {
        todo!("Implement using engine.submit_recv()")
    }
}
```

### Step 5: High-Level Cache Interface

```rust
// src/cache.rs
use crate::{CacheKey, NodeId, ValueLocation, CacheWorker, NodeInfo};
use anyhow::Result;
use std::sync::Arc;

/// High-level distributed cache interface
pub struct DistributedCache {
    worker: Arc<CacheWorker>,
    /// Connection to master for metadata
    master_client: MasterClient,
}

impl DistributedCache {
    pub async fn new(config: WorkerConfig, master_addr: &str) -> Result<Self> {
        // Initialize RDMA worker
        let worker = Arc::new(CacheWorker::new(config)?);
        
        // Connect to master
        let master_client = MasterClient::connect(master_addr).await?;
        
        // Register with master
        master_client.register(worker.get_node_info()).await?;
        
        Ok(Self { worker, master_client })
    }
    
    /// Get a value - fetches from local or remote
    pub async fn get(&self, key: &CacheKey) -> Result<Option<Vec<u8>>> {
        // 1. Check local cache first
        {
            let local_index = self.worker.local_index.read();
            if let Some(&(gpu_idx, block_idx)) = local_index.get(key) {
                // Read from local GPU memory
                return Ok(Some(self.read_local_block(gpu_idx, block_idx)?));
            }
        }
        
        // 2. Query master for location
        let location = match self.master_client.lookup(key).await? {
            Some(loc) => loc,
            None => return Ok(None),
        };
        
        // 3. If on this node (shouldn't happen), read locally
        if location.node_id == self.worker.node_id {
            return Ok(Some(self.read_from_location(&location)?));
        }
        
        // 4. Fetch from remote node using RDMA
        self.fetch_from_remote(key, &location).await
    }
    
    /// Fetch value from remote node
    async fn fetch_from_remote(
        &self,
        key: &CacheKey,
        location: &ValueLocation,
    ) -> Result<Option<Vec<u8>>> {
        // Get remote node info
        let remote_info = self.master_client.get_node_info(location.node_id).await?;
        
        // Ensure we have a peer group for this node
        if !self.worker.peer_groups.read().contains_key(&location.node_id) {
            self.worker.register_remote_node(&remote_info)?;
        }
        
        // Allocate local buffer
        let (block_idx, local_location) = self.worker.allocate_block(0)
            .ok_or_else(|| anyhow::anyhow!("No free blocks"))?;
        
        // Request remote node to RDMA write to us
        let data = self.worker.request_remote_value(
            location.node_id,
            remote_info.rdma_addresses[0].clone(),
            key.clone(),
        ).await?;
        
        Ok(Some(data))
    }
    
    /// Put a value into the cache
    pub async fn put(&self, key: CacheKey, value: &[u8]) -> Result<()> {
        // 1. Allocate local block
        let (block_idx, location) = self.worker.allocate_block(0)
            .ok_or_else(|| anyhow::anyhow!("No free blocks"))?;
        
        // 2. Copy value to GPU memory
        self.write_to_block(0, block_idx, value)?;
        
        // 3. Update local index
        self.worker.local_index.write().insert(key.clone(), (0, block_idx));
        
        // 4. Register with master
        self.master_client.register_key(key, location).await?;
        
        Ok(())
    }
    
    fn read_local_block(&self, gpu_idx: usize, block_idx: usize) -> Result<Vec<u8>> {
        let pool = &self.worker.memory_pools[gpu_idx];
        let block = &pool.blocks[block_idx];
        
        let mut data = vec![0u8; block.length as usize];
        cuda_memcpy_d2h(&mut data, pool.memory.ptr(), block.offset as usize)?;
        Ok(data)
    }
    
    fn write_to_block(&self, gpu_idx: usize, block_idx: usize, data: &[u8]) -> Result<()> {
        let pool = &self.worker.memory_pools[gpu_idx];
        let block = &pool.blocks[block_idx];
        
        cuda_memcpy_h2d(pool.memory.ptr(), block.offset as usize, data)?;
        Ok(())
    }
    
    fn read_from_location(&self, location: &ValueLocation) -> Result<Vec<u8>> {
        // Find the GPU with this memory region
        for (gpu_idx, pool) in self.worker.memory_pools.iter().enumerate() {
            if pool.mr_descriptor.ptr == location.mr_descriptor.ptr {
                let block_idx = (location.offset / pool.blocks[0].length) as usize;
                return self.read_local_block(gpu_idx, block_idx);
            }
        }
        Err(anyhow::anyhow!("Location not found locally"))
    }
}
```

---

## 🔄 Complete Data Flow: Remote Get

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         GET("key123") FLOW                                  │
└────────────────────────────────────────────────────────────────────────────┘

1. Client calls cache.get("key123") on Node A
   │
2. Node A checks local index → Not found locally
   │
3. Node A queries Master → ValueLocation { node: B, mr: {...}, offset: 1024 }
   │
4. Node A allocates local receive buffer
   │  └── buffer registered with RDMA (mr_descriptor available)
   │
5. Node A sends ReadRequest to Node B (via RDMA Send/Recv):
   │  {
   │    request_id: 42,
   │    key: "key123",
   │    response_location: { node: A, mr: {...}, offset: 2048 }
   │  }
   │
6. Node B receives request, looks up local value
   │
7. Node B performs RDMA WRITE directly to Node A's GPU memory:
   │  ┌─────────────────────────────────────────────────────────────┐
   │  │  fi_writemsg(ep, msg, FI_REMOTE_CQ_DATA)                   │
   │  │    src: Node B GPU @ offset 1024                            │
   │  │    dst: Node A GPU @ offset 2048 (from response_location)   │
   │  │    imm_data: 0x1 (signals completion)                       │
   │  └─────────────────────────────────────────────────────────────┘
   │
   │  Data flows: B.GPU → B.NIC → Network → A.NIC → A.GPU
   │              (GPUDirect RDMA - zero copy!)
   │
8. Node A's NIC writes data directly to GPU memory
   │  Node A's CQ receives completion with imm_data
   │
9. Node B sends ReadResponse to Node A (success)
   │
10. Node A reads value from local GPU buffer
```

---

## 🎯 Key Integration Points

### 1. Memory Registration (Critical for RDMA)

```rust
// All memory that will be accessed remotely MUST be registered
let (handle, descriptor) = engine.register_memory_allow_remote(
    gpu_memory.ptr(),
    size,
    Device::Cuda(CudaDeviceId(gpu_id)),
)?;

// Share `descriptor` with remote nodes so they can write to you
// descriptor contains: ptr (virtual address) + [(addr, rkey), ...] per domain
```

### 2. Immediate Data for Signaling

```rust
// Use imm_data to signal completion without extra messages
let request = TransferRequest::Single(SingleTransferRequest {
    // ...
    imm_data: Some(request_id as u32), // Remote gets this in CQ
    // ...
});

// Remote side receives:
// TransferCompletionEntry::ImmData(request_id)
```

### 3. Batch Operations for Throughput

```rust
// For high throughput, submit many transfers and wait with counters
let counter = Arc::new(AtomicI64::new(0));
let err_counter = Arc::new(AtomicI64::new(0));

for request in batch {
    engine.submit_transfer_atomic(request, counter.clone(), err_counter.clone())?;
}

// Wait for all to complete
let expected = batch.len() as i64;
while counter.load(Ordering::Acquire) < expected {
    std::hint::spin_loop();
}
```

---

## 📋 Full Example: Cargo.toml

```toml
[package]
name = "rdma-kv-cache"
edition = "2024"

[dependencies]
fabric-lib = { path = "../fabric-lib", features = ["tokio"] }
cuda-lib = { path = "../cuda-lib" }
thread-lib = { path = "../thread-lib" }

anyhow = "1.0"
bytes = "1.0"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
postcard = "1.0"
parking_lot = "0.12"
tracing = "0.1"
rand = "0.8"
```

---

## 🚀 Performance Tips

1. **Pre-register peer groups**: Call `add_peer_group()` during initialization for known nodes
2. **Reuse memory regions**: Don't register/unregister frequently
3. **Use atomic counters**: For batch operations instead of callbacks
4. **Pin worker threads**: Use the topology detection to place threads on correct NUMA nodes
5. **Multiple NICs**: Use `DomainGroupRouting::RoundRobinSharded` to aggregate bandwidth

Would you like me to elaborate on any specific part, such as the master node implementation, error handling, or a working test example?

```plaintext
┌──────────────────────────────────────────────────────────────────────────┐
│                           MASTER NODE                                     │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                     Metadata Store                                  │  │
│  │  HashMap<Key, ValueLocation>                                        │  │
│  │                                                                     │  │
│  │  ValueLocation {                                                    │  │
│  │      node_id: u32,                                                  │  │
│  │      mr_descriptor: MemoryRegionDescriptor,  // For RDMA access    │  │
│  │      offset: u64,                                                   │  │
│  │      length: u64,                                                   │  │
│  │  }                                                                  │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                    Control Plane (gRPC/TCP)
                    ┌───────────────┴───────────────┐
                    │               │               │
                    ▼               ▼               ▼
┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐
│     WORKER NODE 0    │ │     WORKER NODE 1    │ │     WORKER NODE 2    │
│                      │ │                      │ │                      │
│  ┌────────────────┐  │ │  ┌────────────────┐  │ │  ┌────────────────┐  │
│  │ Value Storage  │  │ │  │ Value Storage  │  │ │  │ Value Storage  │  │
│  │ (GPU/CPU Mem)  │  │ │  │ (GPU/CPU Mem)  │  │ │  │ (GPU/CPU Mem)  │  │
│  │                │  │ │  │                │  │ │  │                │  │
│  │ RDMA-Registered│  │ │  │ RDMA-Registered│  │ │  │ RDMA-Registered│  │
│  └────────────────┘  │ │  └────────────────┘  │ │  └────────────────┘  │
│          │           │ │          │           │ │          │           │
│  ┌────────────────┐  │ │  ┌────────────────┐  │ │  ┌────────────────┐  │
│  │ TransferEngine │  │ │  │ TransferEngine │  │ │  │ TransferEngine │  │
│  └────────────────┘  │ │  └────────────────┘  │ │  └────────────────┘  │
│          │           │ │          │           │ │          │           │
│      RDMA NIC        │ │      RDMA NIC        │ │      RDMA NIC        │
└──────────┬───────────┘ └──────────┬───────────┘ └──────────┬───────────┘
           │                        │                        │
           └────────── RDMA Network (Data Plane) ────────────┘
```

```rust
// src/types.rs
use bytes::Bytes;
use fabric_lib::api::{DomainAddress, MemoryRegionDescriptor};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::Arc;
use parking_lot::RwLock;

/// Unique identifier for each worker node
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct NodeId(pub u32);

/// Key type for the cache (could be a hash or structured key)
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct CacheKey(pub Bytes);

/// Location of a value in the distributed cache
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ValueLocation {
    /// Which node stores this value
    pub node_id: NodeId,
    /// RDMA memory region descriptor for remote access
    pub mr_descriptor: MemoryRegionDescriptor,
    /// Offset within the memory region
    pub offset: u64,
    /// Length of the value in bytes
    pub length: u64,
}

/// Metadata about a node's RDMA endpoint
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NodeInfo {
    pub node_id: NodeId,
    /// RDMA address for this node (one per domain/NIC)
    pub rdma_addresses: Vec<DomainAddress>,
    /// Number of NICs per GPU
    pub nets_per_gpu: u8,
}

/// Master's view of the cluster
pub struct ClusterMetadata {
    /// Key to location mapping
    pub key_locations: RwLock<HashMap<CacheKey, ValueLocation>>,
    /// Node info for RDMA connections
    pub nodes: RwLock<HashMap<NodeId, NodeInfo>>,
}
```

```rust
// src/worker/mod.rs
use anyhow::Result;
use fabric_lib::{
    TransferEngine, TransferEngineBuilder, TransferCallback,
    detect_topology,
    api::{
        DomainAddress, MemoryRegionHandle, MemoryRegionDescriptor,
        SingleTransferRequest, TransferRequest, DomainGroupRouting,
        PeerGroupHandle, SmallVec,
    },
};
use cuda_lib::{Device, CudaDeviceMemory, CudaDeviceId};
use std::collections::HashMap;
use std::ffi::c_void;
use std::num::NonZeroU8;
use std::ptr::NonNull;
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, AtomicI64, Ordering};
use parking_lot::RwLock;

/// Configuration for the cache worker
pub struct WorkerConfig {
    pub node_id: NodeId,
    pub gpu_ids: Vec<u8>,
    pub nets_per_gpu: usize,
    pub block_size: usize,      // Size of each cache block
    pub num_blocks: usize,      // Number of blocks per GPU
}

/// A block of memory that can store a cache value
pub struct CacheBlock {
    pub offset: u64,
    pub length: u64,
    pub in_use: bool,
}

/// Memory pool for a single GPU
pub struct GpuMemoryPool {
    pub gpu_id: u8,
    pub memory: CudaDeviceMemory,
    pub mr_handle: MemoryRegionHandle,
    pub mr_descriptor: MemoryRegionDescriptor,
    pub blocks: Vec<CacheBlock>,
    pub free_blocks: Vec<usize>,
}

/// The RDMA-enabled cache worker
pub struct CacheWorker {
    pub node_id: NodeId,
    pub config: WorkerConfig,
    pub engine: Arc<TransferEngine>,
    pub memory_pools: Vec<GpuMemoryPool>,
    /// Mapping of remote nodes to their peer group handles
    pub peer_groups: RwLock<HashMap<NodeId, PeerGroupHandle>>,
    /// Local key-value index
    pub local_index: RwLock<HashMap<CacheKey, (usize, usize)>>, // (gpu_idx, block_idx)
}

impl CacheWorker {
    /// Initialize the RDMA cache worker
    pub fn new(config: WorkerConfig) -> Result<Self> {
        // Step 1: Detect hardware topology
        let topo_groups = detect_topology()?;
        
        // Step 2: Build TransferEngine with selected GPUs
        let mut builder = TransferEngineBuilder::default();
        for &gpu_id in &config.gpu_ids {
            let topo = topo_groups
                .iter()
                .find(|t| t.cuda_device == gpu_id)
                .ok_or_else(|| anyhow::anyhow!("GPU {} not found", gpu_id))?;
            
            let domains: Vec<_> = topo.domains
                .iter()
                .take(config.nets_per_gpu)
                .cloned()
                .collect();
            
            let worker_cpu = topo.cpus[0];
            let uvm_cpu = topo.cpus[1];
            
            builder.add_gpu_domains(gpu_id, domains, worker_cpu, uvm_cpu);
        }
        let engine = Arc::new(builder.build()?);
        
        // Step 3: Allocate and register memory pools
        let mut memory_pools = Vec::new();
        for &gpu_id in &config.gpu_ids {
            let pool = Self::create_memory_pool(
                &engine,
                gpu_id,
                config.block_size,
                config.num_blocks,
            )?;
            memory_pools.push(pool);
        }
        
        Ok(Self {
            node_id: config.node_id,
            config,
            engine,
            memory_pools,
            peer_groups: RwLock::new(HashMap::new()),
            local_index: RwLock::new(HashMap::new()),
        })
    }
    
    /// Create a memory pool for a GPU
    fn create_memory_pool(
        engine: &TransferEngine,
        gpu_id: u8,
        block_size: usize,
        num_blocks: usize,
    ) -> Result<GpuMemoryPool> {
        use cuda_lib::rt::cudaSetDevice;
        
        // Set CUDA device
        cudaSetDevice(gpu_id as i32)?;
        
        // Allocate GPU memory
        let total_size = block_size * num_blocks;
        let memory = CudaDeviceMemory::device(total_size)?;
        
        // Register with RDMA engine (allow remote access)
        let (mr_handle, mr_descriptor) = engine.register_memory_allow_remote(
            memory.ptr(),
            total_size,
            Device::Cuda(CudaDeviceId(gpu_id)),
        )?;
        
        // Initialize block tracking
        let blocks: Vec<_> = (0..num_blocks)
            .map(|i| CacheBlock {
                offset: (i * block_size) as u64,
                length: block_size as u64,
                in_use: false,
            })
            .collect();
        let free_blocks: Vec<_> = (0..num_blocks).collect();
        
        Ok(GpuMemoryPool {
            gpu_id,
            memory,
            mr_handle,
            mr_descriptor,
            blocks,
            free_blocks,
        })
    }
    
    /// Get node info for registration with master
    pub fn get_node_info(&self) -> NodeInfo {
        NodeInfo {
            node_id: self.node_id,
            rdma_addresses: self.get_all_domain_addresses(),
            nets_per_gpu: self.engine.nets_per_gpu().get(),
        }
    }
    
    /// Get RDMA addresses for all domains
    fn get_all_domain_addresses(&self) -> Vec<DomainAddress> {
        // The main address represents this node
        // In practice, you'd collect addresses from all domains
        vec![self.engine.main_address()]
    }
    
    /// Allocate a block for storing a value
    pub fn allocate_block(&self, gpu_idx: usize) -> Option<(usize, ValueLocation)> {
        let pool = &self.memory_pools[gpu_idx];
        
        // Find a free block (simplified - real impl needs proper locking)
        let block_idx = pool.free_blocks.first().copied()?;
        let block = &pool.blocks[block_idx];
        
        let location = ValueLocation {
            node_id: self.node_id,
            mr_descriptor: pool.mr_descriptor.clone(),
            offset: block.offset,
            length: block.length,
        };
        
        Some((block_idx, location))
    }
    
    /// Register a remote node for RDMA access
    pub fn register_remote_node(&self, remote_info: &NodeInfo) -> Result<()> {
        // Build address list for peer group
        // Each entry in SmallVec corresponds to one local domain
        let addrs: Vec<SmallVec<DomainAddress>> = remote_info.rdma_addresses
            .iter()
            .map(|addr| {
                let mut sv = SmallVec::new();
                sv.push(addr.clone());
                sv
            })
            .collect();
        
        // Add peer group to engine
        let handle = self.engine.add_peer_group(
            addrs,
            Device::Cuda(CudaDeviceId(self.config.gpu_ids[0])),
        )?;
        
        self.peer_groups.write().insert(remote_info.node_id, handle);
        Ok(())
    }
}
```

```rust
// src/worker/rdma_ops.rs
impl CacheWorker {
    /// Fetch a value from a remote node using RDMA
    /// This is a ONE-SIDED operation - no involvement from remote CPU!
    pub async fn fetch_remote_value(
        &self,
        remote_location: &ValueLocation,
        local_gpu_idx: usize,
        local_block_idx: usize,
    ) -> Result<()> {
        let local_pool = &self.memory_pools[local_gpu_idx];
        let local_block = &local_pool.blocks[local_block_idx];
        
        // Build the transfer request
        // Note: fabric-lib uses RDMA WRITE (push model)
        // For a "pull" model (remote read), we need coordination
        
        // Option 1: Have the remote node push data to us
        // Option 2: Use Send/Recv for coordination + Write for data
        
        // Here's the RDMA WRITE approach (remote node pushes to us):
        let request = TransferRequest::Single(SingleTransferRequest {
            // Source: remote node's memory
            src_mr: MemoryRegionHandle::new(
                NonNull::new(remote_location.mr_descriptor.ptr as *mut c_void).unwrap()
            ),
            src_offset: remote_location.offset,
            length: remote_location.length,
            imm_data: None,
            // Destination: our local memory
            dst_mr: local_pool.mr_descriptor.clone(),
            dst_offset: local_block.offset,
            domain: DomainGroupRouting::RoundRobinSharded {
                num_shards: self.engine.nets_per_gpu(),
            },
        });
        
        // Submit and wait for completion
        self.engine.submit_transfer_async(request).await?;
        
        Ok(())
    }
    
    /// Push a local value to a remote node
    pub fn push_to_remote(
        &self,
        local_gpu_idx: usize,
        local_block_idx: usize,
        remote_location: &ValueLocation,
    ) -> Result<tokio::sync::oneshot::Receiver<Result<()>>> {
        let local_pool = &self.memory_pools[local_gpu_idx];
        let local_block = &local_pool.blocks[local_block_idx];
        
        // Create completion channel
        let (tx, rx) = tokio::sync::oneshot::channel();
        
        // Build RDMA WRITE request
        let request = TransferRequest::Single(SingleTransferRequest {
            src_mr: local_pool.mr_handle,
            src_offset: local_block.offset,
            length: local_block.length.min(remote_location.length),
            imm_data: Some(0x1), // Signal completion to remote
            dst_mr: remote_location.mr_descriptor.clone(),
            dst_offset: remote_location.offset,
            domain: DomainGroupRouting::RoundRobinSharded {
                num_shards: self.engine.nets_per_gpu(),
            },
        });
        
        // Submit with callback
        let callback = TransferCallback {
            on_done: Box::new(move || {
                let _ = tx.send(Ok(()));
                Ok(())
            }),
            on_error: Box::new(move |e| {
                // tx already moved, can't send error
                Err(format!("RDMA transfer failed: {:?}", e))
            }),
        };
        
        self.engine.submit_transfer(request, callback)?;
        
        Ok(rx)
    }
    
    /// Fetch with atomic counter for batch operations
    pub fn fetch_batch_atomic(
        &self,
        requests: Vec<(ValueLocation, usize, usize)>, // (remote, local_gpu, local_block)
        completion_counter: Arc<AtomicI64>,
        error_counter: Arc<AtomicI64>,
    ) -> Result<()> {
        for (remote_location, local_gpu_idx, local_block_idx) in requests {
            let local_pool = &self.memory_pools[local_gpu_idx];
            let local_block = &local_pool.blocks[local_block_idx];
            
            let request = TransferRequest::Single(SingleTransferRequest {
                src_mr: local_pool.mr_handle,
                src_offset: local_block.offset,
                length: remote_location.length,
                imm_data: None,
                dst_mr: remote_location.mr_descriptor.clone(),
                dst_offset: remote_location.offset,
                domain: DomainGroupRouting::RoundRobinSharded {
                    num_shards: self.engine.nets_per_gpu(),
                },
            });
            
            self.engine.submit_transfer_atomic(
                request,
                completion_counter.clone(),
                error_counter.clone(),
            )?;
        }
        
        Ok(())
    }
    
    /// Wait for batch completion using spin-wait
    pub fn wait_batch_completion(
        completion_counter: &AtomicI64,
        expected: i64,
    ) {
        while completion_counter.load(Ordering::Acquire) < expected {
            std::hint::spin_loop();
        }
    }
}
```

```rust
// src/protocol.rs
use serde::{Deserialize, Serialize};

/// Messages sent between nodes
#[derive(Serialize, Deserialize)]
pub enum CacheMessage {
    /// Request to read a value - receiver should RDMA write back
    ReadRequest {
        request_id: u64,
        key: CacheKey,
        /// Where to write the response (requestor's memory)
        response_location: ValueLocation,
    },
    
    /// Acknowledgment that write is complete
    ReadResponse {
        request_id: u64,
        success: bool,
        error: Option<String>,
    },
    
    /// Request to write a value
    WriteRequest {
        request_id: u64,
        key: CacheKey,
        /// Where the value is (sender's memory) - receiver will RDMA read
        /// OR sender can push directly
        value_location: ValueLocation,
    },
    
    /// Node registration
    Register(NodeInfo),
}

// src/worker/protocol.rs
impl CacheWorker {
    /// Handle incoming message (via Send/Recv)
    pub async fn handle_message(&self, msg: CacheMessage) -> Result<Option<CacheMessage>> {
        match msg {
            CacheMessage::ReadRequest { request_id, key, response_location } => {
                // Look up local value
                let local_index = self.local_index.read();
                if let Some(&(gpu_idx, block_idx)) = local_index.get(&key) {
                    drop(local_index);
                    
                    // RDMA WRITE the value to requestor's memory
                    let rx = self.push_to_remote(gpu_idx, block_idx, &response_location)?;
                    rx.await??;
                    
                    return Ok(Some(CacheMessage::ReadResponse {
                        request_id,
                        success: true,
                        error: None,
                    }));
                }
                
                Ok(Some(CacheMessage::ReadResponse {
                    request_id,
                    success: false,
                    error: Some("Key not found".to_string()),
                }))
            }
            
            CacheMessage::WriteRequest { request_id, key, value_location } => {
                // Allocate local block
                if let Some((block_idx, local_location)) = self.allocate_block(0) {
                    // Could either:
                    // 1. Have sender push to us (preferred for RDMA)
                    // 2. We fetch from sender
                    
                    // Store in local index
                    self.local_index.write().insert(key, (0, block_idx));
                }
                Ok(None)
            }
            
            _ => Ok(None),
        }
    }
    
    /// Request a value from a remote node (full protocol)
    pub async fn request_remote_value(
        &self,
        remote_node_id: NodeId,
        remote_node_addr: DomainAddress,
        key: CacheKey,
    ) -> Result<Vec<u8>> {
        // 1. Allocate local buffer for response
        let (block_idx, response_location) = self.allocate_block(0)
            .ok_or_else(|| anyhow::anyhow!("No free blocks"))?;
        
        // 2. Send read request (using Send/Recv messaging)
        let request = CacheMessage::ReadRequest {
            request_id: rand::random(),
            key,
            response_location,
        };
        
        // Serialize and send via RDMA Send
        let data = postcard::to_vec(&request)?;
        self.send_message(remote_node_addr.clone(), &data).await?;
        
        // 3. Wait for response message
        let response_data = self.recv_message().await?;
        let response: CacheMessage = postcard::from_bytes(&response_data)?;
        
        // 4. Read data from local buffer
        let pool = &self.memory_pools[0];
        let block = &pool.blocks[block_idx];
        
        // Copy from GPU to host
        let mut result = vec![0u8; block.length as usize];
        cuda_memcpy_d2h(&mut result, pool.memory.ptr(), block.offset as usize)?;
        
        Ok(result)
    }
    
    /// Low-level send message
    async fn send_message(&self, dest: DomainAddress, data: &[u8]) -> Result<()> {
        // Need a registered host buffer for Send/Recv
        // This would be allocated during worker init
        todo!("Implement using engine.submit_send_async()")
    }
    
    /// Low-level receive message
    async fn recv_message(&self) -> Result<Vec<u8>> {
        todo!("Implement using engine.submit_recv()")
    }
}
```

```rust
// src/cache.rs
use crate::{CacheKey, NodeId, ValueLocation, CacheWorker, NodeInfo};
use anyhow::Result;
use std::sync::Arc;

/// High-level distributed cache interface
pub struct DistributedCache {
    worker: Arc<CacheWorker>,
    /// Connection to master for metadata
    master_client: MasterClient,
}

impl DistributedCache {
    pub async fn new(config: WorkerConfig, master_addr: &str) -> Result<Self> {
        // Initialize RDMA worker
        let worker = Arc::new(CacheWorker::new(config)?);
        
        // Connect to master
        let master_client = MasterClient::connect(master_addr).await?;
        
        // Register with master
        master_client.register(worker.get_node_info()).await?;
        
        Ok(Self { worker, master_client })
    }
    
    /// Get a value - fetches from local or remote
    pub async fn get(&self, key: &CacheKey) -> Result<Option<Vec<u8>>> {
        // 1. Check local cache first
        {
            let local_index = self.worker.local_index.read();
            if let Some(&(gpu_idx, block_idx)) = local_index.get(key) {
                // Read from local GPU memory
                return Ok(Some(self.read_local_block(gpu_idx, block_idx)?));
            }
        }
        
        // 2. Query master for location
        let location = match self.master_client.lookup(key).await? {
            Some(loc) => loc,
            None => return Ok(None),
        };
        
        // 3. If on this node (shouldn't happen), read locally
        if location.node_id == self.worker.node_id {
            return Ok(Some(self.read_from_location(&location)?));
        }
        
        // 4. Fetch from remote node using RDMA
        self.fetch_from_remote(key, &location).await
    }
    
    /// Fetch value from remote node
    async fn fetch_from_remote(
        &self,
        key: &CacheKey,
        location: &ValueLocation,
    ) -> Result<Option<Vec<u8>>> {
        // Get remote node info
        let remote_info = self.master_client.get_node_info(location.node_id).await?;
        
        // Ensure we have a peer group for this node
        if !self.worker.peer_groups.read().contains_key(&location.node_id) {
            self.worker.register_remote_node(&remote_info)?;
        }
        
        // Allocate local buffer
        let (block_idx, local_location) = self.worker.allocate_block(0)
            .ok_or_else(|| anyhow::anyhow!("No free blocks"))?;
        
        // Request remote node to RDMA write to us
        let data = self.worker.request_remote_value(
            location.node_id,
            remote_info.rdma_addresses[0].clone(),
            key.clone(),
        ).await?;
        
        Ok(Some(data))
    }
    
    /// Put a value into the cache
    pub async fn put(&self, key: CacheKey, value: &[u8]) -> Result<()> {
        // 1. Allocate local block
        let (block_idx, location) = self.worker.allocate_block(0)
            .ok_or_else(|| anyhow::anyhow!("No free blocks"))?;
        
        // 2. Copy value to GPU memory
        self.write_to_block(0, block_idx, value)?;
        
        // 3. Update local index
        self.worker.local_index.write().insert(key.clone(), (0, block_idx));
        
        // 4. Register with master
        self.master_client.register_key(key, location).await?;
        
        Ok(())
    }
    
    fn read_local_block(&self, gpu_idx: usize, block_idx: usize) -> Result<Vec<u8>> {
        let pool = &self.worker.memory_pools[gpu_idx];
        let block = &pool.blocks[block_idx];
        
        let mut data = vec![0u8; block.length as usize];
        cuda_memcpy_d2h(&mut data, pool.memory.ptr(), block.offset as usize)?;
        Ok(data)
    }
    
    fn write_to_block(&self, gpu_idx: usize, block_idx: usize, data: &[u8]) -> Result<()> {
        let pool = &self.worker.memory_pools[gpu_idx];
        let block = &pool.blocks[block_idx];
        
        cuda_memcpy_h2d(pool.memory.ptr(), block.offset as usize, data)?;
        Ok(())
    }
    
    fn read_from_location(&self, location: &ValueLocation) -> Result<Vec<u8>> {
        // Find the GPU with this memory region
        for (gpu_idx, pool) in self.worker.memory_pools.iter().enumerate() {
            if pool.mr_descriptor.ptr == location.mr_descriptor.ptr {
                let block_idx = (location.offset / pool.blocks[0].length) as usize;
                return self.read_local_block(gpu_idx, block_idx);
            }
        }
        Err(anyhow::anyhow!("Location not found locally"))
    }
}
```

```plaintext
┌────────────────────────────────────────────────────────────────────────────┐
│                         GET("key123") FLOW                                  │
└────────────────────────────────────────────────────────────────────────────┘

1. Client calls cache.get("key123") on Node A
   │
2. Node A checks local index → Not found locally
   │
3. Node A queries Master → ValueLocation { node: B, mr: {...}, offset: 1024 }
   │
4. Node A allocates local receive buffer
   │  └── buffer registered with RDMA (mr_descriptor available)
   │
5. Node A sends ReadRequest to Node B (via RDMA Send/Recv):
   │  {
   │    request_id: 42,
   │    key: "key123",
   │    response_location: { node: A, mr: {...}, offset: 2048 }
   │  }
   │
6. Node B receives request, looks up local value
   │
7. Node B performs RDMA WRITE directly to Node A's GPU memory:
   │  ┌─────────────────────────────────────────────────────────────┐
   │  │  fi_writemsg(ep, msg, FI_REMOTE_CQ_DATA)                   │
   │  │    src: Node B GPU @ offset 1024                            │
   │  │    dst: Node A GPU @ offset 2048 (from response_location)   │
   │  │    imm_data: 0x1 (signals completion)                       │
   │  └─────────────────────────────────────────────────────────────┘
   │
   │  Data flows: B.GPU → B.NIC → Network → A.NIC → A.GPU
   │              (GPUDirect RDMA - zero copy!)
   │
8. Node A's NIC writes data directly to GPU memory
   │  Node A's CQ receives completion with imm_data
   │
9. Node B sends ReadResponse to Node A (success)
   │
10. Node A reads value from local GPU buffer
```

```rust
// All memory that will be accessed remotely MUST be registered
let (handle, descriptor) = engine.register_memory_allow_remote(
    gpu_memory.ptr(),
    size,
    Device::Cuda(CudaDeviceId(gpu_id)),
)?;

// Share `descriptor` with remote nodes so they can write to you
// descriptor contains: ptr (virtual address) + [(addr, rkey), ...] per domain
```

```rust
// Use imm_data to signal completion without extra messages
let request = TransferRequest::Single(SingleTransferRequest {
    // ...
    imm_data: Some(request_id as u32), // Remote gets this in CQ
    // ...
});

// Remote side receives:
// TransferCompletionEntry::ImmData(request_id)
```

```rust
// For high throughput, submit many transfers and wait with counters
let counter = Arc::new(AtomicI64::new(0));
let err_counter = Arc::new(AtomicI64::new(0));

for request in batch {
    engine.submit_transfer_atomic(request, counter.clone(), err_counter.clone())?;
}

// Wait for all to complete
let expected = batch.len() as i64;
while counter.load(Ordering::Acquire) < expected {
    std::hint::spin_loop();
}
```

```toml
[package]
name = "rdma-kv-cache"
edition = "2024"

[dependencies]
fabric-lib = { path = "../fabric-lib", features = ["tokio"] }
cuda-lib = { path = "../cuda-lib" }
thread-lib = { path = "../thread-lib" }

anyhow = "1.0"
bytes = "1.0"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
postcard = "1.0"
parking_lot = "0.12"
tracing = "0.1"
rand = "0.8"
```

