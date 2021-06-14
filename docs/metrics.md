# Performance Metrics

**WORK IN PROGRESS**

You've written a program and you want to know if the performance is good. How do you know? It depends on the hardware! There are a few common ways to measure performance, and the metric you use to evaluate your program's performance will depend on the most likely bottleneck: memory or compute. Keep in mind that if you run a tiny problem on the GPU (e.g. sum two numbers) your bottleneck will be the cost of transferring data back and forth (i.e. latency).

# Occupancy

`Occupancy = # resident warps / max possible resident warps`

Limiting factors:
- Thread group size
- Thread Group Shared Memory (TGSM) usage
- Number of registers per thread group

HW units have a limit of:
- Max # of thread groups in flight
- Available shared memory divided by resident thread groups
- Max threads per thread group

Example:
- HW shader unit supports 8 thread groups, has 32KB of shared memory, and 1024 threads
- If you dispatch TG of 128 threads and uses 24KB shared memory you can only have 1 TG resident, which is probably bad.

Keep in mind that 100% occupancy doesn't always translate to superior performance. See [this doc](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#occupancy) for an explanation of why.

# Processing Power (FLOPS)

If a shader has lots of arithmetic complexity it will likely have a bottleneck in instructions/ALUs.

`Cores * Clock Speed * FLOPs/cycle`

**Example: GTX 1080**
- 2560 cores (80 streaming multiprocessors (SM) * 32 cores per SM)
- 1.607 GHz base clock speed
- 2 FLOPs/cycle (fused multiply-add (FMA))
- 2560 * 1.607 * 2 = 8227.84 GFLOPS (single precision)

**Example: Intel Core i7-8650U**
- 4 cores
- 4.2 GHz boosted clock speed
- 32 FLOPs/cycle (assuming AVX-512, 512bits/32bit=16 * 2 (FMA3) = 32 for SP)
- 537 GFLOPS (single precision). 500 billion operations per second!

Hitting peak numbers tends to be more complicated in practice: consider specialized instructions (e.g. AVX-512) and hardware units (TensorCores).

Theoretical limit that is essentially impossible to reach, but serves as an indicator of the raw compute power. Why impossible to reach?
- Data needs to be available.
- All cores need to be fully saturated with work.
- No pipeline stalls or other waits.

**Note**:
- Consumer GPUs typically have much FP64 throughput. Why? 64-bit is typically used only in HPC/professional applications, so it is not efficient for gaming cards to pack tons of these units.
- Older architectures did not have high FP16 throughput, but newer architectures often have double the throughput of FP32. Even if FP16 compute has lower FLOPS, it saves a ton in memory. If memory bound this may still be a win.

# Memory Bandwidth (GB/s)

Shader is simple but accesses lots of data? Probably memory bound. Now we need to know the hardware's theoretical memory bandwidth.

Calculate peak theoretical bandwidth: memory speed * bus width

**Example: G80 GPU**
- Bus width: 384-bit
- Speed: 900 MHz DDR (GDDR3 = double data rate = 2X speed) = 1800 MHz = 1.8 Gbps
- Bandwidth: (384 * 1.8) / 8 = 86.4 GBps

**Example: NVIDIA GeForce GTX 1080**
- Bus Width: 256-bit
- Speed: 1389 MHz (1958 MHz boosted); GDDR5X (4X); 10 Gbps memory speed
- Bandwidth: (256 * 10) / 8 = 320 GBps

Remember that hardware typically publishes *theoretical* limits, and you are very unlikely to reach these limits in most cases. However, they provide a reasonable way to evaluate how effectively your program utilizes the hardware.

# Resources

- [GFLOPS in an ML model](http://www.hpcuserforum.com/presentations/Wisconsin2017/HPDLCookbook4HPCUserForum.pdf)
- https://en.wikipedia.org/wiki/FLOPS
- Wikipedia will often list theoretical throughput/memory bandwidth for all GPUs