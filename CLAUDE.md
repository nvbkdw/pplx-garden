# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

pplx-garden is Perplexity AI's open source RDMA-based communication library for LLM inference. It provides:
- **fabric-lib**: RDMA TransferEngine library supporting NVIDIA ConnectX-7 and AWS EFA
- **p2p-all-to-all**: Point-to-point MoE (Mixture of Experts) dispatch/combine kernels

## Build Commands

### Development Environment (Docker)
```bash
docker build -t pplx-garden-dev - < docker/dev.Dockerfile
./scripts/run-docker.sh
```

### Rust Build
```bash
cargo build --release                        # Build all crates
cargo build --release --bin fabric-debug     # Build benchmark binary only
cargo test                                   # Run Rust tests
```

### Python Wheel
```bash
export TORCH_CMAKE_PREFIX_PATH=$(python3 -c "import torch; print(torch.utils.cmake_prefix_path)")
python3 -m build --wheel
python3 -m pip install dist/*.whl
```

### Python Tests
```bash
pytest tests/                                        # All tests
pytest tests/ -m "not (ci_2gpu or ci_4gpu)"          # Single GPU tests
pytest tests/ -m ci_2gpu                             # Tests requiring 2 GPUs
pytest tests/ -m ci_4gpu                             # Tests requiring 4 GPUs
pytest tests/ -m kernel                              # Kernel tests only
pytest tests/ -m fabric                              # Fabric tests only
```

### Linting
```bash
ruff check python/ tests/ benchmarks/
ruff format python/ tests/ benchmarks/
cargo clippy
```

## Architecture

### Provider Abstraction (`fabric-lib/src/`)
The RDMA library abstracts over two network providers via the `RdmaDomain` trait:
- **EFA** (`efa/`): AWS Elastic Fabric Adapter using libfabric
- **Verbs** (`verbs/`): InfiniBand verbs (ConnectX-7) using libibverbs

Key components:
- `TransferEngine`: High-level async transfer API with callbacks
- `FabricEngine`: Core engine managing multiple workers
- `Worker`: Per-NIC worker handling RDMA operations
- `DomainGroup`: Groups NICs for multi-NIC aggregation per GPU

### All-to-All Kernels (`p2p-all-to-all/`)
CUDA kernels for MoE dispatch/combine operations:
- `a2a-kernels/`: CUDA kernel implementations (dispatch_send, dispatch_recv, combine_send, combine_recv)
- `AllToAllContext`: Rust orchestration layer managing GPU workspace and worker threads
- Uses NVLink for intra-node, RDMA for inter-node transfers

### Python Bindings
- `python-ext/`: PyO3 bindings exposing Rust to Python
- `python/pplx_garden/`: Python package
  - `kernels/`: High-level all-to-all interfaces
  - `native/`: Low-level bindings to CUDA memory and p2p operations
  - `distributed/`: Process group and NCCL utilities

## System Requirements
- Linux Kernel 5.12+ (DMA-BUF support)
- CUDA 12.8+
- libfabric, libibverbs, GDRCopy
- `SYS_PTRACE` and `SYS_ADMIN` capabilities for `pidfd_getfd`
- RDMA network with GPUDirect RDMA support

## Running fabric-debug Benchmark
```bash
# Server
fabric-debug 0,1,2,3,4,5,6,7 2

# Client (use address printed by server)
fabric-debug 0,1,2,3,4,5,6,7 2 fe80xxxx
```
Arguments: `[GPU indices] [NICs per GPU] [server address for client]`

## Running All-to-All Benchmark
```bash
NUM_NODES=... NODE_RANK=... MASTER_IP=...
python3 -m benchmarks.bench_all_to_all \
    --world-size $((NUM_NODES * 8)) --nets-per-gpu 2 \
    --init-method=tcp://$MASTER_IP:29500 \
    --node-rank=$NODE_RANK --nvlink=8
```
Remove `--nvlink` for RDMA-only mode.
