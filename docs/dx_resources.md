# DirectX Resources & Views <!-- omit in toc -->

Resources in Direct3D store the data that shaders use to perform their computations. There are two types of resources in D3D: buffers and textures. Buffers are contiguous arrays of individually-accessed data elements. Textures are structured containers of *texels* (think images) that are typically *sampled* (hardware-accelerated interpolation at locations that may be between individual texels). This doc currently only covers buffers -- they're far more common in ML compute applications -- but I may add details on textures at some point.

- [Views](#views)
- [Buffers](#buffers)
  - [Typed Buffers](#typed-buffers)
  - [Structured Buffers](#structured-buffers)
  - [Append/Consume Buffers](#appendconsume-buffers)
  - [Raw Buffers](#raw-buffers)
  - [Constant Buffers](#constant-buffers)
  - [Texture Buffers](#texture-buffers)
- [Performance Considerations](#performance-considerations)

# Views

Shaders never read or write directly to resources. Instead, you create a *view* of a resource that imposes some logical structure and properties on how the resource is accessed. 

There are three types of views in D3D12:
- **Shader Resource View** (SRV): a read-only view of a resource that may or may not support random access
- **Unordered Access View** (UAV): a read-write view of a resource that supports random access
- **Constant Buffer View** (CBV): a read-only view of a *buffer* resource with data that is uniform across a dispatch (usually placed in lower-latency memory)

When deciding between SRVs or UAVs you ideally only use UAVs if the shader will write to the resource; an SRV is a promise to the hardware that you won't modify the resource, so it may allow some optimizations. This is just a general guideline, however, and in practice the overhead of using a UAV on a resource you don't intend to write to may not be noticeable.

# Buffers

Each type of view has parameters, and the manner in which you configure the view parameters leads to different *kinds* of resource views. Below is a summary of the kinds of buffer views you can create.

| Kind            | HLSL Declaration        | Type | Format               | Flags          | Stride | Counter   |
| --------------- | ----------------------- | ---- | -------------------- | -------------- | ------ | --------- |
| Typed           | Buffer                  | SRV  | Multiple             | `NONE`         | 0      | -         |
| Typed           | RWBuffer                | UAV  | Multiple<sup>1</sup> | `NONE`         | 0      | `nullptr` |
| Structured      | StructuredBuffer        | SRV  | `UNKNOWN`            | `NONE`         | > 0    | -         |
| Structured      | RWStructuredBuffer      | UAV  | `UNKNOWN`            | `NONE`         | > 0    | `nullptr` |
| Append          | AppendStructuredBuffer  | UAV  | `UNKNOWN`            | `NONE`         | > 0    | Yes       |
| Consume         | ConsumeStructuredBuffer | UAV  | `UNKNOWN`            | `NONE`         | > 0    | Yes       |
| Raw             | ByteAddressBuffer       | SRV  | `R32_TYPELESS`       | `SRV_FLAG_RAW` | 0      | -         |
| Raw             | RWByteAddressBuffer     | UAV  | `R32_TYPELESS`       | `UAV_FLAG_RAW` | 0      | `nullptr` |
| Constant Buffer | cbuffer                 | CBV  | -                    | -              | -      | -         |
| Texture Buffer  | tbuffer                 | SRV  | Multiple             | `NONE`         | 0      | -         |

<sup>1</sup> [See allowed formats for typed UAV loads](https://docs.microsoft.com/en-gb/windows/win32/direct3d12/typed-unordered-access-view-loads). All hardware supports at least R32_FLOAT, R32_UINT, and R32_SINT, but the other formats are optional and support must be checked at runtime.

## Typed Buffers

A [typed buffer](https://docs.microsoft.com/en-us/windows/win32/direct3d11/direct3d-11-advanced-stages-cs-resources#structured-buffer) is a view into a buffer resource that allows you to access data with a declared storage [format](https://docs.microsoft.com/en-us/windows/win32/api/dxgiformat/ne-dxgiformat-dxgi_format). For example, you may indicate the buffer comprises contiguous 32-bit floating-point values (R32_FLOAT), groups of 4x 8-bit unsigned integers (R8G8B8A8_UINT), and many more. The benefit to attaching a type to the view of the buffer is that you don't need to rewrite the shader code for each possible data type that you may bind. As an example, you can declare a `Buffer<float>` and bind a buffer filled with single-precision data (R32_FLOAT) or with half-precision data (R16_FLOAT); the hardware will automatically and efficiently handle the format conversion.

Example (shader uses FP32, but data in the buffer may be FP16 or FP32):
```cpp
Buffer<float> input;
RWBuffer<float> output;

[numthreads(1, 1, 1)]
void CSMain(uint3 dtid : SV_DispatchThreadID)
{
    output[dtid.x] = input[dtid.x] * 2;
}
```

A useful application of typed buffers is FP16 compute scenarios: some hardware does not support `float16_t` in shader code, but this same hardware *may* support typed UAV loads with R16_FLOAT. This means you can bind buffers filled with half-precision data to shaders and do single-precision arithmetic. You won't get the performance benefits of half-precision, but you *do* get the bandwidth benefits and can run on older hardware without failing.

Note that it is perfectly valid to declare a type in the view that doesn't match the data written into the buffer: for example, the buffer may have UINT32 written into it but the view of the buffer may specify UINT16 with twice as many elements. This is analogous to a C++ reinterpret_cast and can sometimes be useful.

One pitfall to be aware of is overflow behavior when using typed buffers. Usually, if you do some computation that results in overflow, integers will wrap (e.g. storing 256 in UINT8 will be 0) and floating-point values will go to infinity. This is not true for typed buffers, and instead writes of elements that have overflowed with respect to their storage type will instead clamp to the numeric limit for that data type (e.g. writing 256 into a UINT8 buffer will write 255).

## Structured Buffers

A [structured buffer](https://docs.microsoft.com/en-us/windows/win32/direct3d11/direct3d-11-advanced-stages-cs-resources#structured-buffer) is a view into a buffer resource that allows you to access data as an array of structures. These structures may comprise a mixture of data types of varying sizes, so you do not specify a DXGI_FORMAT when creating the view. Instead, you must specify DXGI_FORMAT_UNKNOWN and set the total size of the structure with the `StructureByteStride` field on the view.

Example (structure is 8 bytes):
```cpp
struct Point
{
    float x;
    float y;
};

RWStructuredBuffer<Point> input;

[numthreads(1, 1, 1)]
void CSMain(uint3 dtid : SV_DispatchThreadID)
{
    input[0].x = -5;
    input[0].y = 32;
}
```

It may not be obvious -- some articles even claim it's not possible -- but you *can* use structured buffers with with primitives types directly (e.g. `StructuredBuffer<float>`). 

Keep in mind that you will not get any hardware format conversions with structured buffers: you're essentially overlaying the structure layout, as declared in HLSL, on top of the raw bytes of the buffer.  The hardware *may* do some form of swizzling to make access more efficient (e.g. for array fields accessed by thread index), but the raw buffer data does no go through any format conversion path like in typed buffers.

## Append/Consume Buffers

[Append](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/sm5-object-appendstructuredbuffer) and [consume](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/sm5-object-consumestructuredbuffer) buffers are views that allow you to access data in a buffer like a stack: you can push items onto the end of an append buffer, and you can pop items off the end of a consume buffer. The data elements in these buffers are structures (just like a standard structured buffer) with a uniform size. The position of where to push and pop items is managed by a second buffer resource that must have a 32-but unsigned integer value (atomic instructions ensure the counter value is updated in a thread-safe manner). This makes append/consume buffers the only view that references *two* resources.

Append/consume buffers are unusual outside of niche rendering applications like [order-independent transparency](https://en.wikipedia.org/wiki/Order-independent_transparency). Usually it's desirable to control the memory access pattern of threads, and treating the data as a stack introduces additional overhead to manage the counter state. [Supposedly](http://www.joshbarczak.com/blog/?p=1260) these buffers exist purely as a way to leverage the global data store (GDS) in AMD architectures that is shared by all thread groups and is *not* global memory.

Below is an example showing float elements being popped off the input buffer and pushed onto the output buffer. The views of the the input and output buffers would counter resources initialized to the size of the respective stacks (e.g. input would likely be positive and output 0).
```cpp
ConsumeStructuredBuffer<float> input;
AppendStructuredBuffer<float> output;

[numthreads(NUM_THREADS, 1, 1)]
void CSMain(uint3 dtid : SV_DispatchThreadID)
{
    float x = input.Consume();
    output.Append(x);
}
```

## Raw Buffers

A raw buffer, also called a [byte-address buffer](https://docs.microsoft.com/en-us/windows/win32/direct3d11/direct3d-11-advanced-stages-cs-resources#byte-address-buffer), is a view into a buffer resource that allows you to access data byte by byte; conceptually it's equivalent to a C++ `std::array<std::byte>`. Unless your shader operates on bytes directly, this means you'll need to manually pack/unpack and reinterpret (e.g. using [asfloat](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-asfloat)) to do anything useful with it. There are some language conveniences to make this easier, however, like [loading and storing with struct type](https://github.com/microsoft/DirectXShaderCompiler/wiki/ByteAddressBuffer-Load-Store-Additions).

One reason to use raw buffers over structured buffers is that it enables you to view the data as a mixture of differently-sized types; elements in a structured buffer must, by definition, have the same layout and size. Additionally, loading from raw buffers *may* be faster than structured and typed buffers on some hardware.

Example (doubles the second 32-bit float in the input buffer)
```cpp
RWByteAddressBuffer input;

[numthreads(1, 1, 1)]
void CSMain(uint3 dtid : SV_DispatchThreadID)
{
    float x = asfloat(input.Load(4)) * 4;
    input.Store(4, asuint(x));
}
```

## Constant Buffers

A [constant buffer](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createconstantbufferview) (cbuffer) is a view into a buffer resource that provides read-only access to frequently accessed data. The key difference between constant buffer views (CBVs) and other read-only resource views (i.e. SRVs) is that constant buffers are subject to size, alignment, and [special packing requirements](https://github.com/microsoft/DirectXShaderCompiler/wiki/Buffer-Packing#constant-buffer-packing). These constraints allow (but don't *require*) the buffers to be kept in lower-latency memory in hardware (e.g. caches).

Below is an example in HLSL that declares two constants (`elementCount`, `alpha`) in a constant buffer:
```cpp
cbuffer constants
{
    uint elementCount;
    float alpha;
};

[numthreads(256, 1, 1)]
void CSMain(uint3 dtid : SV_DispatchThreadID)
{
    if (dtid.x < elementCount)
    {
        ...
    }
}
```

## Texture Buffers

A [texture buffer](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-constants) (tbuffer) is, like a constant buffer, a view into a buffer (or texture!) resource that provides read-only access to frequently accessed data; however, if constant buffers are optimized for linear reads then texture buffers are optimized for scattered reads. This view is more commonly applied to buffer resources, but it is (unlike all other buffer views) valid as a view of a texture resource. 

Texture butters are created using an SRV with a a valid DXGI_FORMAT and number of elements.

Below is an example in HLSL that declares two constants (`elementCount`, `alpha`) in a constant buffer:
```cpp
tbuffer constants
{
    uint elementCount;
    float alpha;
};

[numthreads(256, 1, 1)]
void CSMain(uint3 dtid : SV_DispatchThreadID)
{
    if (dtid.x < elementCount)
    {
        ...
    }
}
```

Typically the underlying resource is a buffer resource, but *can* be a texture resource.

Must be SRV in a descriptor table; cannot use root descriptors.

# Performance Considerations

You'll have noticed that many resource views can be used to accomplish the same task, and you may be wondering which one you *should* use to get the best performance. For example, is it better to used `Buffer<float>` or `StructuredBuffer<float>`? How important is it to bind resources as SRVs if they're read only? Unfortunately, the D3D API docs and specs don't shed light on any of the potential performance tradeoffs of different resource views; it's an implementation detail! The best advice is to profile on the hardware you're targeting (if that's even possible).

If you're curious, however, you can check out Sebastian Aaltonen's [perftest](https://github.com/sebbbi/perftest) repo to get an idea of how different resource views behave on different hardware.