# Chunking in o1js

This RFC describes how we plan to integrate chunking in o1js, making it accessible to developers. The original chunking implementation has been described in detail in [RFC0002](./0002-chunking.md).

## Summary

Chunking in our proof system enhances its capacity, allowing it to handle circuits with a significantly higher number of constraints — the previous limit of 2^16 constraints is expanded to 2^32. This advancement empowers developers using o1js to create and prove more complex computations, paving the way for a broader spectrum of innovative applications and use cases. For instance, consider the process of hashing a single data blob using SHA256, which demands approximately 5000 constraints. Under the old constraint limit, a user could perform about 13 SHA256 hashes within a single circuit. The enhanced limit greatly reduces such restrictions, offering far more flexibility and potential in the types of computations that can be executed in a single operation.

## Motivation

The motivation, assessment criteria, and exit criteria can be found in the original [Chunking PRD](https://www.notion.so/o1labs/Chunked-Commitments-PRD-890c836529c545f082a3eeecd3d4f510).

This RFC describes the process of exposing and implementing chunking in o1js, making it available to o1js developers. Chunking itself has already been implemented in Kimchi and Pickles, the remaining work only requires exposing the new functionality to o1js via the bindings layer.

## Detailed design

Currently, the circuit writing API in o1js is exposed via the `SmartContract` and `ZkProgram` API. Under the hood, we use the OCaml bindings (js_of_ocaml) as an [interface to Pickles](https://github.com/o1-labs/o1js-bindings/blob/43fa328c4ef4a225e1343a7f26fc3d85adf67b21/ocaml/lib/pickles_bindings.ml#L577-L624).

In the TypeScript layer, [compiling circuits](https://github.com/o1-labs/o1js/blob/16e66f9fba51ac4832691c3fc7a52bae60a08fd7/src/lib/proof_system.ts#L683-L688) calls into the bindings layer via `Pickles.compile`, passing in the individual branches (rules or methods) of the smart contract or ZkProgram and returning prover artifacts.

In order to make chunking work in o1js, we have to modify the `Pickles.compile` to accept the number of chunks that are required in order to construct the proof. The Pickles [test](https://github.com/MinaProtocol/mina/blob/develop/src/lib/pickles/test/chunked_circuits/test_chunked_circuits.ml) serves as an example on how to modify the `Pickles.compile` function to prove larger circuits and specify the amount of chunks.

Calculating how many chunks are needed to prove a circuit depends on the amount of constraints used in a circuit. Before passing the parameter to the function and compiling the circuit, we have to calculate the amount of chunks needed. Luckily, o1js already has a function that can inspect methods and count the amount of constraints, we get this information by calling `.analyzeMethods()` on the `ZkProgram` or `SmartContract`.

### Technical Implementation

The implementation should not require and API changes for the developer, all changes happen internally and hidden from the public facing API. A naive overview of the changes:

1. Adjusting OCaml bindings to Pickles - [pickles_bindings.ml](https://github.com/o1-labs/o1js-bindings/blob/1db0524c5c7c93f0df9db3fb3bdfa0414bdaf90b/ocaml/lib/pickles_bindings.ml#L563)

   - Modifying the `pickles_compile` function to accept an additional parameter `num_chunks`, which will then be passed into `Pickles.compile_promise`. `num_chunks` will be calculates in o1js directly.

2. Adapting the compile function in [proof_system.ts](https://github.com/o1-labs/o1js/blob/0d82ad1d67417c72b06fc8456a0a2b54bcc46652/src/lib/proof_system.ts)

   - `compileProgram` function should accept a new argument `numChunks` which represents the amount of chunks needed to compile the circuit
   - `numChunks` inside of `compileProgram` will then be passed into `Pickles.compile` via the bindings layer

3. Before calling into `compileProgram`, the number of chunks must be calculated. This can be done by utilizing the `analyzeMethods` function which exists on both `SmartContract` and `ZkProgram`.
   - For `ZkProgram`, this change can be done in `proof_systems.ts` and the `ZkProgram` function
   - For `SmartContract`, the change is needed in [zkapp.ts](https://github.com/o1-labs/o1js/blob/0d82ad1d67417c72b06fc8456a0a2b54bcc46652/src/lib/zkapp.ts) - in the `SmartContract.compile` method.

- ## Changing the type definitions of the bindings to represent the change in the `pickles_compile` function

## Test plan and functional requirements

1. Testing Goals and Objectives:

- o1js can compile and prove circuits that are bigger than 2^16 constraints, up to ideally 2^32 (the new limit introduced by chunking)
- A proof using a larger circuit limit can be deployed to and verified on a testnet
- Circuits that exceed the old limit of 2^16 constraints can be recursively verified in each other

2. Testing Approach:

- multiple tests in o1js that test the new limits of the circuits. Tests in Kimchi and Pickles already exists, they can be used as [examples](https://github.com/MinaProtocol/mina/blob/develop/src/lib/pickles/test/chunked_circuits/test_chunked_circuits.ml)

3. Testing Scope:

- Proofs can be constructed and verified locally as well as deployed to a testnet.

## Drawbacks

Proving larger circuits will increase proving time and memory consumption. Given the memory limit in most browsers and WASM, this could cause problems for users trying to prove large circuits in a browser.

## Rationale and alternatives

Not exposing Chunking and continuing to rely on recursion for large computations. This adds a significant overhead for developers and adds complexity of applications.

## Prior art

Chunking working in [OCaml and Pickles](https://github.com/MinaProtocol/mina/blob/develop/src/lib/pickles/test/chunked_circuits/test_chunked_circuits.ml) directly.

## Unresolved questions

While trying to prototype Chunking in o1js, I came up with the following unresolved questions. The tests can be found in [this](https://github.com/MinaProtocol/mina/tree/chunking_rfc_unresolved) branch.

1 Changing the loop that constructs the circuit from `2^17` (roughly `2^16` constraints) iterations to `2^18` iterations (or `2^17` constraints) and also adjusting `num_chunks: 4` and `override_wrap_domain:N1` compiles the program correctly, generates a proof _but_ doesn't verify the proof:

```sh
Uncaught exception:

  (Pickles.verify dlog_check)

Raised at Base__Error.raise in file "src/error.ml" (inlined), line 8, characters 14-30
Called from Base__Or_error.ok_exn in file "src/or_error.ml", line 84, characters 17-32
Called from Dune__exe__Invalid_proof in file "src/lib/pickles/test/chunked_circuits/invalid_proof.ml", line 92, characters 2-9
```

This can be reproduced by running `dune exec ./invalid_proof.exe` inside of the test folder `src/lib/pickles/test/chunked_circuits`.

Is this behaviors expected? If so, why and how can we prevent the construction of invalid circuits and proofs in o1js? Changing `num_chunks` and `override_wrap_domain` results in an (expected) compilation error - so the chosen values seem to be logical and correct.

On the other hand, setting `num_chunks:8` and the loop iteration to `2^19` compiles the circuit and yields a valid proof. (see `valid_proof.ml`)

2 What is the maximum allowed number chunks? From my understanding, setting iteration to `2^20` and `num_chunks:16` requires that the wrap domain must be `N3`, but it only accepts `N0 | N1 | N2`. Setting the wrap domain to anything `<= N2` yields an error. How do we deal with large number chunks and constraints? See `large_wrap_domain.ml` for this case.

3 Running a small MVP of Chunking in o1js through the bindings yields the following error: `there are not enough random rows to achieve zero-knowledge (expected: 3, got: 2)`. It seems like some values aren't passed correctly. This also needs to be resolved during implementation.
