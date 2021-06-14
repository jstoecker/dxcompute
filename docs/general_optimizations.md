# General GPU Optimization Strategies <!-- omit in toc -->

**WORK IN PROGRESS**

This doc covers general strategies and tips for writing efficient compute shaders that are not specific to (but may be influenced by) any particular hardware architecture. The optimizations are listed (very roughly) by priority, but keep in mind lots of small optimizations may add up to a large performance gain.

- [Avoid Round Trips to the CPU](#avoid-round-trips-to-the-cpu)
- [Avoid Divergent Branching](#avoid-divergent-branching)
- [Global Memory: Leverage Coalesced Access](#global-memory-leverage-coalesced-access)
- [Shared Memory: Avoid Bank Conflicts](#shared-memory-avoid-bank-conflicts)
- [Prefer Smaller Thread Groups](#prefer-smaller-thread-groups)
- [Reduce Register Pressure](#reduce-register-pressure)
- [Eliminate Barriers for Data Dependencies Within a Wave](#eliminate-barriers-for-data-dependencies-within-a-wave)
- [Reduce Instruction Overhead](#reduce-instruction-overhead)
- [Minor Optimizations](#minor-optimizations)
- [Learning Resources](#learning-resources)

# Avoid Round Trips to the CPU

Transferring data between the CPU and GPU is slow but unavoidable; however, it's not as big deal if this can be batched or streamed in a way that keeps both processors busy at the same time. You generally want to keep as much work on the GPU as possible until it's finished. For example, executing the operations in a machine learning model can be extremely inefficient if only a small set of the operations are accelerated by the GPU: bouncing between the CPU and GPU will incur "round trip" costs that force CPU/GPU synchronization and result in idle processors. This often makes it desirable to perform calculations that are not ideal for GPUs to avoid the round trip.

# Avoid Divergent Branching

Conditional logic is very common in CPU programs, and this is often implemented using a jump instruction. CPUs also have sophisticated [branch predictors](https://en.wikipedia.org/wiki/Branch_predictor) to improve pipelining. GPUs, on the other hand, have neither jump instructions nor branch predictors. Each SIMD processor evaluates each branch taken by 1 more threads; the SIMD lanes are deactivated for threads that don't participate in a particular branch. In the worst case each thread takes a different branch and the cost of branching is the sum of the time to evaluate all branches. Ouch!

```cpp
// BAD: both branches will need to be executed with half the threads idle in each branch
if (threadId % 2 == 0) {
  // branch 1
} else {
  // branch 2
}
```

On the flip side, branching is only expensive if threads in a warp take different branches: if all threads share the same branch then the extra branches are *not* executed. One way to optimize shaders that require branching is to arrange the data such that threads in each warp will take the same branch.

Another technique that may be useful is using a lookup table for the possible results of each branch. This isn't always feasible, but if the branch conditions can be converted into an index and the branch result stored as a resource/constant then the cost of divergence can be eliminated.

# Global Memory: Leverage Coalesced Access

Global memory access from threads within a warp will have their requests coalesced into fewer memory transactions, which is significantly faster. The best way to take advantage of this is to ensure threads in a warp access nearby elements from global memory. The CUDA guide has a great explanation of [coalesced memory access](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#coalesced-access-to-global-memory), but this technique is generally relevant to other hardware architectures as well.

# Shared Memory: Avoid Bank Conflicts

Shared memory is like a user-controlled cache: your thread groups can populate it with data from global memory that is repeatedly accessed by different threads. Using shared memory can greatly improve efficiency in memory-bound shaders but you should be aware of [bank conflicts](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#shared-memory).

Accessing the same bank from **multiple threads** will create a bank conflict and accesses will be serialized. When possible, threads should limit their access to the same banks (e.g. thread ID 0 always uses bank 0, thread ID 1 always uses bank 1).

# Prefer Smaller Thread Groups

There is no one-size-fits-all answer to how large your thread groups and dispatches should be, but in general you should try to balance occupancy and resource usage (shared memory & registers). The CUDA guide gives some [excellent tips](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#thread-and-block-heuristics) here.

For instance, a single thread group can have 1024 threads, but it's usually not a good idea to max this out when you can use more thread groups instead. You want to spread work across all shader units, so a general tip is to have at least a few thread groups per shader unit. Having a few thread groups per shader unit will improve overall utilization of the processors when memory requests stall.

Wave sizes vary across hardware, but common sizes are 16, 32, and 64. If your shader runs on a wide range of hardware you'll probably want your thread groups to have sizes that are a multiple of 64.

# Reduce Register Pressure

GPUs need to hide latency by scheduling lots of work: when a thread group makes a memory request it stalls and another thread group can start executing. This context switch must be lightning fast, and we know memory access is slow, so the execution state of a thread group cannot simply be written to memory (the thread group is stalled waiting for such a memory request!). Instead, the thread group's state is written to registers. To effectively hide memory latency a single SM may need to cycle through several thread groups, so GPUs have very large register files to store all this state. There's a limit, of course, and the more registers a compute shader requires the fewer thread groups can be in flight before exhausting the register file.

Where possible, you should eliminate local variables or state that needs to be preserved *while memory access is pending* (state that doesn't need to be preserved when a stall occurs is irrelevant). This will allow your compute shader to have higher occupancy.

# Eliminate Barriers for Data Dependencies Within a Wave

There's no need to synchronize threads within a wave because, by definition, all threads in a wave execute the same instructions simultaneously on a SIMD processor. It's safe to write instructions without barriers when depending on the output of threads that are in the same wave. 

# Reduce Instruction Overhead

Reads, writes, and arithmetic in the core computation are unavoidable. However, some instructions in a computer shader can be eliminated or simplified. For example, loops can introduce significant instruction overhead. Consider the following loop:

```cpp
float result = 0;
for (int i = 0; i < 5; i++)
   result += input[i * 2];
```

Each iteration of the loop involves updating the index `i` and doing some address arithmetic (`i*2`). This loop can be *unrolled* as follows:

```cpp
float result = 0;
[unroll] for (int i = 0; i < 5; i++)
   result += input[i * 2];
```

Which should generate could that is logically similar to the following:

```cpp
float result = 0;
result += input[0];
result += input[2];
result += input[4];
result += input[6];
result += input[8];
```

# Minor Optimizations

[Avoid Division and Modulo](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#division-and-modulo-operations). It's well-known to graphics programmers that division is expensive, but sometimes you just can't avoid it. There are, however, tricks you can employ in some cases. A common trick is to precompute the divisor as a reciprocal and multiply instead.

# Learning Resources

- [DirectCompute Optimizations and Best Practices](https://on-demand.gputechconf.com/gtc/2010/presentations/S12312-DirectCompute-Pre-Conference-Tutorial.pdf). Slides from around the time DirectCompute was first released, but most of the optimizations still apply today.
- [CUDA C++ Best Practices Guide](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html). Includes several tips on memory and instruction optimizations. Many of these tips are relevant to different generations and architectures across vendors, but do keep in mind this is primarily targeting NVIDIA GPUs.