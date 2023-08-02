# RFC title

GPU based implementations of costly crypto algorithms.

## Summary

A rust library implementing gpu based acceleration for computing the most commonly used crypto
algorithms, it can be integrated in kimchi and other projects that make use of them with the
pasta curves and their fields.

## Motivation

One of the main components of kimchi's runtime are the multi scalar multiplication (MSM), fast fourier
transform (FFT), and polynomial operations.
This library will accelerate that process doing the computation in a gpu, it will also serve as a reference
point to measure the improvements that gpu computations can provide us.
This will build on top of an already existing implementation, and thus some of the design choices will be
based on the fact that the particular decision is cheap to implement because it is already implemented.
The implementation will mainly target WebGPU with the goal of working on any platform.

## Detailed design

The implementation will be based in the wgpu graphics API and thus the WGSL shader language. The library
will consist of 3 crates:
    - arithmetic
    - shader
    - runner
The runner crate will be the one intended for use inside another projects, it will be added to kimchi as
an optional, non-default dependency, and potentially promoted in the future to default.

### arithmetic

This is a crate implementing general field arithmetic, general curve arithmetic and specifically the
pasta curves and their field. It uses no dependencies and avoids hardware features that are unsupported
by GPUs in general and WGSL in particular.
The code is also written in a way optimized for GPUs, but it can still be used in any architecture, while
it will be generally slower that other cpu implementations it can be used for things like tests.
The crate also contains code to generate constants that are used by the shader while the code itself doesn't
have to be run in the shader.
The process of compiling rust to WGSL in particular as shown issues, thus it was decided to write WGSL
through a more manual process, this crate will have its scope limited to generate constants and will
potentially change into a code generation utility that will help create WGSL shaders.

### shader

The shader is a small crate that will be compiled to WGSL, the code is very simple, mostly calls to
functions from the arithmetic crate, compilation takes a significant time and thus the compiled
shader will be saved and the runner will used that precompiled shader by default.
This one will probably be removed when we move to directly writing WGSL, it may still be useful to
do a vulkan/spirv implementation in the future, but that isn't in the scope of this rfc.

### runner

This is library that will be used by end-users of the crate, it is the part that will run in the CPU
and will talk to the GPU, it will expose a simple API that will ask for the points that make the base
of the MSM and then perform the initialization and provide some struct that can be used to send scalar
and get back the MSM of them and the base sent in the initialization.
A more advance interface could just offer some wrapper types for polynomials that only allow operations
between the polynomials and msm which will look the same but internally do the gpu calls, this would
allow to optimize for memory transfers by just moving to and from the gpu when needed with intermediate
results never leaving the gpu.

```rust
let points: Vec<Point> = todo!();
let gpu_msm = GpuMsm::new(points);
let scalars: Vec<Scalar> = todo!();
let msm = gpu_msm.msm(scalars);
```

## Test plan and functional requirements

Given that the goal is more to see the kind of performance gains we can obtain, the testing can be
limited to asserting that the result obtained is the same as our current CPU implementation.
Before this becomes the default implementation to be used in production we will require a considerable
amount of reviewing and additional test considering that the arithmetic crate is an implementation
from scratch of the basic cryptographic building blocks.
There are already some unit tests and some tests using formal verification, but the formal verification
tests take a while to run a probably can't cover all cases.
At the end we should have a benchmark comparing cpu and gpu implementations.

## Drawbacks

- There is the possibility of something outside of our control not working, like the nvidia Vulkan drivers
  when I tried to use Vulkan instead of wgpu.
- There could be no significant gains, the Vulkan implementations was able to run ~3 times faster than
  multithreaded arkworks MSM and ~10 times faster than single threaded arkworks while using integrated
  graphics.
  The move to WGSL should reduce the performance of arithmetic about 4 times, but we can probably expect
  discrete GPUs to perform more than 4 times better, also the ratio is hard to measure but part of the
  time will be due to memory operations that shouldn't be affected like arithmetic.

## Rationale and alternatives

There are also FPGA/ASIC implementations, they would provide higher performance in comparison, but the cost
in both time and money will be significatively higher even compared to doing the GPU implementation from
scratch, also GPUs are widely available to most users and have the possibility of being used from the browser.

## Prior art

We currently use a CPU implementation but in terms of hardware acceleration we don't have anything in use,
and if we add the requirement of working in the browser there may not even be anything available that
works with our stack.
We have some groth16 implementation that makes use of GPU and claims a 3x improvement, but I don't know much
about it, and regardless we no longer use groth16.

## Unresolved questions
