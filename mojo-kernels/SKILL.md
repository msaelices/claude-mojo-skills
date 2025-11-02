---
name: mojo-kernels
description: Guide for writing high-performance GPU kernels in Mojo using the MAX framework, covering element-wise operations, reductions, shared memory patterns, and optimization techniques
---

# Mojo GPU Kernel Development

This skill guides you in writing efficient GPU kernels in Mojo using Modular's MAX framework. Use when implementing custom GPU operations, optimizing compute-intensive workloads, or working with SIMD and parallel algorithms.

## Core Concepts

### MAX GPU Kernel Architecture

Mojo GPU kernels execute on GPU devices via the MAX framework. Key differences from CUDA:
- Regular function syntax (no special decorators)
- Compile-time parameters for type safety: `fn kernel[dtype: DType](data: DeviceBuffer[dtype])`
- Explicit address space annotations
- **LayoutTensor**: Preferred high-level tensor abstraction

### Essential Imports

```mojo
from max.gpu import barrier, global_idx, thread_idx, warp, WARP_SIZE
from max.gpu.host import DeviceContext, DeviceBuffer
from max.layout import LayoutTensor
from max.algorithm import elementwise, reduce_launch
from max.math import exp, log, sqrt, tanh, ceildiv
```

## Common Kernel Patterns

### Pattern 1: Element-wise Operations

**Use case**: Operations processing each element independently (activations, arithmetic)

**Structure**:
```mojo
fn elementwise_kernel[dtype: DType](
    output: DeviceBuffer[dtype],
    input: DeviceBuffer[dtype],
    size: Int
):
    var tid = global_idx.x
    if tid >= UInt(size):
        return
    output[tid] = operation(input[tid])
```

**Launch**:
```mojo
var block_size = 256
var grid_size = ceildiv(size, block_size)
ctx.enqueue_function_checked[elementwise_kernel, dtype](
    output, input, size,
    grid_dim=grid_size,
    block_dim=block_size
)
```

### Pattern 2: Warp-level Reduction

**Use case**: Fast reductions within a warp (32 threads)

**Available operations**: `warp.sum()`, `warp.max()`, `warp.min()`, `warp.broadcast()`, `warp.shuffle()`

**Example**:
```mojo
fn warp_reduce_kernel[dtype: DType](
    output: DeviceBuffer[dtype],
    input: DeviceBuffer[dtype],
    size: Int
):
    var tid = global_idx.x
    var value = input[tid] if tid < UInt(size) else 0

    # Warp-level reduction (implicit synchronization)
    var result = warp.sum(value)

    # First thread in warp writes result
    if thread_idx.x % UInt(WARP_SIZE) == 0:
        output[tid // UInt(WARP_SIZE)] = result
```

### Pattern 3: Block-level Reduction with Shared Memory

**Use case**: Reductions across entire thread block (up to 1024 threads)

**Key points**:
- Use `stack_allocation[dtype, size, AddressSpace.SHARED]()`
- Call `barrier()` after writes, before reads
- Two-level: warp reduction, then across warps

**Example**:
```mojo
fn block_reduce_kernel[dtype: DType, block_size: Int](
    output: DeviceBuffer[dtype],
    input: DeviceBuffer[dtype],
    size: Int
):
    alias num_warps = block_size // WARP_SIZE
    var shared = stack_allocation[num_warps, dtype, AddressSpace.SHARED]()

    var tid = global_idx.x
    var local_tid = thread_idx.x
    var warp_id = local_tid // UInt(WARP_SIZE)
    var lane_id = local_tid % UInt(WARP_SIZE)

    # Load and warp-reduce
    var value = input[tid] if tid < UInt(size) else 0
    var warp_sum = warp.sum(value)

    # First thread in each warp writes to shared memory
    if lane_id == 0:
        shared[Int(warp_id)] = warp_sum
    barrier()

    # First warp reduces across warp results
    if local_tid < UInt(num_warps):
        var final_value = shared[Int(local_tid)]
        final_value = warp.sum(final_value)
        if local_tid == 0:
            output[global_idx.x // UInt(block_size)] = final_value
```

### Pattern 4: Shared Memory for Data Reuse

**Use case**: Operations with neighbor access (convolution, stencil operations)

**Example** (1D stencil):
```mojo
fn stencil_kernel[dtype: DType, block_size: Int, radius: Int](
    output: DeviceBuffer[dtype],
    input: DeviceBuffer[dtype],
    size: Int
):
    alias tile_size = block_size + 2 * radius
    var shared = stack_allocation[tile_size, dtype, AddressSpace.SHARED]()

    var tid = global_idx.x
    var local_tid = thread_idx.x

    # Cooperative loading into shared memory
    if tid < UInt(size):
        shared[Int(local_tid + UInt(radius))] = input[tid]

    # Load halo elements
    if local_tid < UInt(radius) and tid >= UInt(radius):
        shared[Int(local_tid)] = input[tid - UInt(radius)]
    if local_tid < UInt(radius) and tid + UInt(block_size) < UInt(size):
        shared[Int(local_tid + UInt(block_size + radius))] = input[tid + UInt(block_size)]

    barrier()

    # Compute with shared memory access
    if tid < UInt(size):
        var result: SIMD[dtype, 1] = 0
        for i in range(-radius, radius + 1):
            result += shared[Int(local_tid) + radius + i]
        output[tid] = result
```

## Device Context Management

Standard pattern for GPU execution:

```mojo
from max.gpu.host import DeviceContext

with DeviceContext() as ctx:
    # Allocate device buffers
    var buffer = ctx.enqueue_create_buffer[DType.float32](size)

    # Copy data to device
    ctx.enqueue_copy(buffer, host_data)

    # Launch kernel
    ctx.enqueue_function_checked[kernel_fn, dtype](
        buffer, size,
        grid_dim=grid_size,
        block_dim=block_size
    )

    # Copy results back
    ctx.enqueue_copy(host_result, buffer)

    # Wait for completion
    ctx.synchronize()
```

## Type System and SIMD

### Compile-time Type Parameters

```mojo
fn typed_kernel[dtype: DType](data: DeviceBuffer[dtype], size: Int):
    # dtype is known at compile time - generates specialized code
    var value = SIMD[dtype, 1](0)
```

### SIMD Operations

```mojo
# Create SIMD vector
var vec = SIMD[DType.float32, 8](0)

# Load/store
vec = input.load[8](offset)
output.store(offset, vec)

# Operations
vec = vec + other_vec
vec = vec * scalar
var sum = vec.reduce_add()
var max_val = vec.reduce_max()

# Type casting
var int_vec = vec.cast[DType.int32]()
```

## Common Activation Functions

Efficient implementations using `@always_inline`:

```mojo
@always_inline
fn relu[dtype: DType](x: SIMD[dtype, 1]) -> SIMD[dtype, 1]:
    return select(x > 0, x, 0)

@always_inline
fn gelu[dtype: DType](x: SIMD[dtype, 1]) -> SIMD[dtype, 1]:
    # Approximation: 0.5 * x * (1 + tanh(sqrt(2/π) * (x + 0.044715 * x^3)))
    var k = SIMD[dtype, 1](0.7978845608)  # sqrt(2/π)
    var a = SIMD[dtype, 1](0.044715)
    var t = k * (x + a * x * x * x)
    return 0.5 * x * (1.0 + tanh(t))

@always_inline
fn silu[dtype: DType](x: SIMD[dtype, 1]) -> SIMD[dtype, 1]:
    # SiLU/Swish: x * sigmoid(x)
    return x / (1.0 + exp(-x))
```

## Performance Optimization

### Occupancy
- **Block size**: 128-256 threads (multiple of 32 for full warps)
- **Grid size**: `ceildiv(total_elements, block_size)`
- Balance registers, shared memory, and thread count

### Memory Coalescing
- Consecutive threads should access consecutive memory addresses
- Stride-1 access pattern: `data[global_idx.x]` ✓
- Strided access: `data[global_idx.x * stride]` ✗ (slower)

### Warp Divergence
- Avoid: `if thread_idx.x % 2 == 0: do_a() else: do_b()`
- Prefer: `result = select(condition, value_a, value_b)`
- Keep control flow uniform within warps

### Shared Memory Bank Conflicts
- 32 banks, 4-byte width
- Avoid: stride-32 access patterns (all threads hit same bank)
- Solution: Pad shared arrays by 1-2 elements

## High-Level MAX Patterns

MAX provides optimized pattern generators:

```mojo
# Element-wise operations
from max.algorithm import elementwise

elementwise[operation, pack_size=8, target="gpu"](
    dims, ctx, input_buffers, output_buffer
)

# Reductions
from max.algorithm import reduce_launch

reduce_launch[
    dtype, reduce_fn, init_value,
    axis, keep_dims, target="gpu"
](shape, ctx, input_buffer, output_buffer)
```

## Best Practices

1. **Use LayoutTensor** for high-level tensor operations over raw pointers
2. **Bounds checking**: Always include `if tid >= size: return`
3. **Compile-time specialization**: Use `@parameter` and `[dtype: DType]` parameters
4. **Warp primitives**: Prefer `warp.sum()` over manual reductions
5. **Barrier placement**: After shared memory writes, before reads
6. **Device context**: Always use `with DeviceContext() as ctx:`
7. **Type safety**: Leverage beartype and Mojo's type system
8. **Grid/block dimensions**: Use `ceildiv()` to avoid missing elements
9. **Initialization**: Zero-initialize shared memory and accumulators
10. **Testing**: Verify across data types and input sizes

## Common Pitfalls

- ❌ Missing `barrier()` after shared memory writes → race conditions
- ❌ Incorrect grid size calculation → missing elements or out-of-bounds
- ❌ Using `if/else` in warp-uniform code → divergence slowdown
- ❌ Stride-32 shared memory access → bank conflicts
- ❌ Not checking `tid >= size` → buffer overruns
- ❌ Forgetting `ctx.synchronize()` → reading incomplete results

## Quick Reference

### Thread Indexing
```mojo
global_idx.x          # Global thread ID
thread_idx.x          # Thread ID within block
global_idx.x // UInt(WARP_SIZE)  # Warp ID (global)
thread_idx.x // UInt(WARP_SIZE)  # Warp ID (local)
thread_idx.x % UInt(WARP_SIZE)   # Lane ID within warp
```

### Synchronization
```mojo
barrier()             # Block-level synchronization
# Warp operations have implicit synchronization
```

### Pattern Selection Guide
- **Element-wise**: No inter-thread communication
- **Warp reduction**: ≤ 32 elements, single value per warp
- **Block reduction**: ≤ 1024 elements, shared memory required
- **Shared memory**: Neighbor access, data reuse, stencils

## Resources

- [MAX Documentation](https://docs.modular.com/max/)
- [Mojo Programming Manual](https://docs.modular.com/mojo/)
- [MAX GPU API Reference](https://docs.modular.com/max/api/python/max/gpu/)
- [MAX Examples](https://github.com/modularml/max/tree/main/examples)

## Version Notes

This skill is based on MAX 24.5+. For latest API changes, consult the [MAX changelog](https://docs.modular.com/max/changelog/).
