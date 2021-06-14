# Shaders

**WORK IN PROGRESS**

| Feature Level                                                                   | Compiler | WDDM                    | Notable features for compute                             |
| ------------------------------------------------------------------------------- | -------- | ----------------------- | -------------------------------------------------------- |
| 5.1                                                                             | FXC      | 2.0 (Windows 10 - 1507) | Supported by all DX12 devices.                           |
| [6.0](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.0) | DXC      | 2.1 (Windows 10 - 1607) | Wave intrinsics; *optional* support for 64-bit integers. |
| [6.1](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.1) | DXC      | 2.3 (Windows 10 - 1709) | -                                                        |
| [6.2](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.2) | DXC      | 2.4 (Windows 10 - 1803) | Explicit support for 16-bit data types.                  |
| [6.3](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.3) | DXC      | 2.5 (Windows 10 - 1809) | -                                                        |
| [6.4](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.4) | DXC      | 2.6 (Windows 10 - 1903) | Packed dot-product intrinsics.                           |
| [6.5](https://microsoft.github.io/DirectX-Specs/d3d/HLSL_ShaderModel6_5.html)   | DXC      | 2.7 (Windows 10 - 2004) | Additional wave intrinsics.                              |
| [6.6](https://microsoft.github.io/DirectX-Specs/d3d/HLSL_ShaderModel6_6.html)   | DXC      | 2.9 (Windows 10 - ?)    | New atomic operations; additional wave intrinsics.       |

# Wave Intrinsics

Compute shaders before model 6 present an execution model that does not reflect grouping of threads into waves. Newer model allows you to write wave-level instructions to leverage the reality of how thread groups are scheduled in hardware.

https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics

# 16-Bit Scalar Types

https://github.com/microsoft/DirectXShaderCompiler/wiki/16-Bit-Scalar-Types

# Packed Dot-Product Intrinsics

https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.4#packed-dot-product-intrinsics

# Resources

- [Compute Shader Stage - D3D11 Spec](https://microsoft.github.io/DirectX-Specs/d3d/archive/D3D11_3_FunctionalSpec.htm#18%20Compute%20Shader%20Stage). Covers aspects of the compute shader stage in the D3D11 API. Ignoring the specific D3D11 API calls, much of this is still relevant in D3D12. Some constraints in this doc no longer exist in D3D12 (e.g. binding slots, hiding of thread waves) and DXBC is superseded by DXIL in newer shader models.
- [Shader Models](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model)
- [Shader Models vs Shader Profiles](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-models)