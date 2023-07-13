# RFC title

GPU based MSM implementation.

## Summary

A rust library implementing gpu based acceleration for computing MSM, it can be integrated in kimchi and
other projects that make use of MSMs with the pasta curves.

## Motivation

One of the main components of kimchi's runtime is the multi scalar multiplication (MSM), it consist of
basically taking a list of points, adding each point to itself N times, and then add all the resulting
points together.
This library will accelerate that process doing the computation in a gpu, it will also serve as a reference
point to measure the improvements that gpu computations can provide us.

This will build on top of an already existing implementation, and thus some of the design choices will be
based on the fact that the particular decision is cheap to implement because it is already implemented.

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

### shader

The shader is a small crate that will be compiled to WGSL, the code is very simple, mostly calls to
functions from the arithmetic crate, compilation takes a significant time and thus the compiled
shader will be saved and the runner will used that precompiled shader by default.

### runner

This is library that will be used by end-users of the crate, it is the part that will run in the CPU
and will talk to the GPU, it will expose a simple API that will ask for the points that make the base
of the MSM and then perform the initialization and provide some struct that can be used to send scalar
and get back the MSM of them and the base sent in the initialization.

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

The main alternative would be to directly write WGSL shaders instead of compiling Rust, would offer more
control over what is running in the GPU but could imply more work and the current approach has the advantage
of being already mostly implemented.
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

While the scope is mostly clear, when this is fully implemented it will open the possibility of implementing
other things in GPU like FFTs and polynomial operations that are not part of this RFC but could be part of a
future RFC depending on the results of this one.
