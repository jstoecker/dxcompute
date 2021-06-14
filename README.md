# GPU Compute with DirectX

**These pages provide a survey of topics related to GPU compute in the DirectX platform with an emphasis on machine learning. This is a work in progress!** 

Most literature related to DirectX compute shaders is scattered across blogs, forum posts, and API docs; it can be challenging for newcomers to get started (especially for those without a background in real-time graphics). Furthermore, the majority of learning resources are focused on CUDA, OpenCL, or compute shaders for rendering purposes. 

The docs here are not a fully self-contained guide, and you will need to explore external resources to become proficient. However, it is my hope that these pages serve as an outline and refer you to useful resources. With that in mind, you should be comfortable with the topics in the first section, *GPU Programming*, before moving on to the others; the rest of the sections are not in any particular order.

**GPU Programming**
- [Compute on GPUs](docs/gpgpu.md)
- [Programming Model](docs/programming_model.md)
- [Modern Architectures](docs/architectures.md)

**DirectX**
- [Overview](docs/directx.md)
- [Resources & Views](docs/dx_resources.md)
- [Compute Shaders](docs/dx_shaders.md)

**Performance**
- [Performance Metrics](docs/metrics.md)
- [General Optimization Strategies](docs/general_optimizations.md)
- [Profiling Tools](docs/profiling_tools.md)

**Machine Learning**
- [Tensor Layouts & Strides](docs/tensor_layouts.md)
- [Data Formats & Quantization](docs/data_formats.md)

<!-- **Algorithms**
- Convolution (depthwise, backprop, winograd)
- GEMM
- Reduction -->