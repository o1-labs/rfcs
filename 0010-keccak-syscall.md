# Keccak Syscalls

## Summary

A MIPS program is a succession of MIPS instructions. Some instructions are
syscall instructions. Only a subset of syscalls have to be implemented for the
OP MIPS zkVM. This document will describe how the syscalls have to be handled
and how the state of the VM must be modified.
A specific attention will be given to the pre-image reading and writing syscalls
which requires some hashing using [Keccak][./keccak.md], therefore giving the
name to this RFC.

The program can request data from the outside world, but instead of using an
RPC, the data will be loaded through an oracle, called the pre-image oracle. You
can find a better description about it in the RFC [Client/Host
integration](./0011-cli-host-integration.md).

## Motivation

We aim to implement the same syscalls in the MIPS zkVM than the current version
that Cannon supports . The behavior must be the same. The VM must read and
write to the same registers.

## Detailed design

Multiple general purpose registers will be used. We will give an alias for each
of them in this document and we will refer to them by their name.

<!-- https://www.dsi.unive.it/~gasparetto/materials/MIPS_Instruction_Set.pdf -->
Here the mapping:
- `zero` - register number 0
- `v0` - register number 2
- `v1` - register number 3
- `a0` - register number 4
- `a1` - register number 5
- `a2` - register number 6
- `a3` - register number 7

This document will also suppose that the zkVM has some pre-defined interface to
read and write the memory and the registers, and the details of the
implementation and the underlying generated constraints will be omitted.
Here provided the environment:
<!-- TODO: add more while implementing -->
- `set_register_value(idx, v)`: set the register number `idx` to the value `v`
- `get_register_value(idx, buffer_v)`: copy the register number `idx` in the variable `buffer_v`
- `set_halted(exit_code)`: constraint halting the program with the exit code `exit_code`.

At the beginning of the instruction, the syscall number is in the register `v0`.

The list of supported syscalls are:
- `sysMmap`, mapped to value `4090`
- `sysBrk`, mapped to value `4045`
- `sysClone`, mapped to value `4120`
- `sysExitGroup`, mapped to value `4246`
- `sysRead`, mapped to value `4003`
- `sysWrite`, mapped to value `4004`
- `sysFcntl`, mapped to value `4055`

Except in the case of the syscall `sysExitGroup`, at the end of the instruction,
the instruction pointer must be increased by a word (32bits) as the next
instruction pointer.

For any other syscall value, the value `0` must be set into the registers `v0`
and `a3`.

An important note is that IO operations are truncated to `4` bytes, meaning that
the register `v0` is at most `4`.
Another important note is that addresses must be aligned at 4 bytes. IO syscalls
**must** support unaligned addresses, i.e. the address `42` can be given even
though it is not a multiple of `4`.
Computing the effective address of an address `addr` can be done using the
bitwise AND operator with `0xFFffFFfc`, which sets the latest two bits to `0`,
i.e. takes the previous multiple of `4` which represents the aligned memory
address. For instance, for the unaligned address `42`, the address `40` must be
used while fetching the memory.

Note that as the syscall instruction must be encoded in a circuit and its
behavior depends on the value in the register `v0`, we must represent its effect
using a polynomial constraining the value `v0`. We can easily do that using a
vanishing polynomial on the values different than the one expected.
For instance, let's suppose `v0` is `sysFnctl` and `sysFnctl` behavior is represented
by the polynomial `P(v0, a0, a1, a3)`, we will use the polynomial:
```
P_syscall_fnctl(v0, a0, a1, a3) = \
  (v0 - 4090) *
  (v0 - 4045) *
  (v0 - 4120) *
  (v0 - 4246) *
  (v0 - 4003) *
  (v0 - 4004) *
  P(v0, a0, a1, a3)
```

to activate the `P_syscall_fnctl` behavior when `v0 = 4055` and deactivate it
when it has a different value. Reduction to polynomials of at most degree `2` can be
done using [quadracization](./0008-kimchi-polynomial-quadracization.md).

We suppose quadracization is going to be used, and therefore, we will not talk
about the degree of the different polynomial constraints, and the detailled
description will stay high level using the API described above, with some
additional tweaks if necessary for a specific syscall behavior.

Regarding the pre-oracle data, it will be loaded as part of the witness given to
the prover.
In the context of generating a proof using the zkVM, the program will first be
run outside of the zkVM, and therefore we know which data and how much data will
be requested by the program.
A new vector of bytes can be added to the Witness structure, mimicking the work
done for the memory, which will keep track of the data requested by the program.
The [CLI](./cli.md) integration will take care of providing and populating
correctly the Witness instance.

Regarding the encoding, we must represent the preimage-key as
a field element of the scalar field of the pairing-friendly curve BN128, which
is 254 bits. The keys are 32 bytes long as it is the result of a KECCAK execution and
therefore is too large to be saved in a field element.
According to the [CLI RFC](./cli.md), the first byte of the hash must be dropped to be
replaced by the request type, leaving therefore 248 bits to be encoded in a
field element, which fits into an element of the scalar field of BN128.
The first byte can be used to encode the table ID the pre-image data are going
to be in.
For the rest of the document, it will be supposed that only type 2 pre-image
keys must be supported, i.e. the table ID will be considered to be a certain
arbitrary constant.

Regarding the the pre-image data, it is important to note that it can be of any
size. The efficiency of a representation and encoding in a lookup table will
depend on the sizes of the data and also its number, and therefore it is
left to the kind of applications the VM is running.
However, there are some optimisations that can be used.
As the key is only 31 bytes, i.e. 248 bits, there are still 5 bits of data that
can be used to encode an index of the data for a specific key. Therefore, for a
given pre-image key, we can encode 2^5 * 31 bytes, i.e. 992 bytes. In terms of
constraints, it will cost an additional constraint to mask the key and get the
corresponding index. Using this encoding trick, and supposing the table can be
of maximum size 2^16, we can encode 2048 different pre-image keys in one table.

In addition to this technique, a wider table can also be used to encode bigger data.

<!-- TODO: a word on the pre-image key as we can only read 4 bytes by 4 bytes,
     we need to save it somewhere -->

### `sysMmap`

<!-- DONE -->

This can be ignored at the moment and implemented as a no-op.
For consistency with Cannon, `v0` must be set to the value of the register `a0`
and `a3` must be set to `0`.

### `sysBrk`

<!-- DONE -->

The register `v0` must be set to `0x400000` and the register `a3` to `0`.

### `sysClone`

<!-- DONE -->

The register `v0` must be set to `1` and the register `a3` to `0`.

### `sysExitGroup`

<!-- DONE -->

The program must halt in this case and `set_halted` must be called with the value `a0`.

### `sysRead`

This syscall is used to get data from the outside world and write in the VM memory.

The register `a0` contains the file descriptor of the input channel, the
register `a1` the address and the `a2` contains the number of bytes that must be
read.

At the end of the call, the register `v0` must contain the number of bytes that
have been read, and `a3` must contain the error code.

Different input channels must be supported.

#### Standard input

<!-- DONE -->

In the case of the standard input, i.e. when `a0` equals `0`, nothing in
particular happens. Therefore, the registers `v0` and `a3` are set to `0`, which
corresponds to no data has been read and no error is raised.

#### Pre-image oracle

When the buffer to be read is the pre-image oracle, i.e. when `a0` equals `5`, a
chunk of the pre-image is read, and copy into the memory.
The pre-image that must be read is determined by the `preimageKey`, and the actual
chunk is given by the preimage offset variable.

The number of bytes that must be read is in the register `a2`.

At the end of the call, the register `v0` must contain the number of bytes that
have been read.


#### Hint response

<!-- DONE -->

When `a0` equals `3` (alias by `fdHintRead`), we simulate we read all data by
setting `v0` to the value contained in the register `a2`. It has no other effect.

#### Default case

<!-- DONE -->

In the case of any other value for the input channel, the register `v0` must be
set to `0xFFffFFff` and the error code `0x9` must be written in the register
`a3`.


### `sysWrite`

This syscall is used to write data into an output channel specified by a file descriptor.
The file descriptor is in the register `a0`. The address of the data is in the
register `a1` and the length in bytes of the data that must be written is in the
register `a2`.
At the end, the actual number of bytes that have been written must be set into
the register `v0` and the error code, if any, must be written in the register
`a3`.

Here the different output channels that must be supported:

#### Standard output

<!-- DONE -->

The standard output has the file descriptor value `1`.

`a2` bytes must be read from `a1` and copied in the standard output.
The number of bytes, i.e. the value in `a2` must be copied into `v0` and the
value `0` must be set in `a3`.

#### Standard error

<!-- DONE -->

The standard error has the file descriptor value `2`.
`a2` bytes must be read from `a1` and copied in the standard error.

The number of bytes, i.e. the value in `a2` must be copied into `v0` and the
value `0` must be set in `a3`.

#### Hint write

<!-- DONE -->

In Cannon, even though the body is not empty, the code has no effect on the
output. It can therefore be ignored.
The number of bytes, i.e. the value in `a2`, must be copied into `v0` and
the value `0` must be set in `a3`.

#### Pre-image Oracle

<!-- DONE -->

The file descriptor `fdPreimageWrite` (value `6`) is used to read a pre-image
key from the memory.
The address to be read is located in the register `a1` and the number of bytes
that has to be read is in the register `a2`, and must be constrained to 4 bytes
as any IO operations.

The bytes are written in a pre-image key variable from right to left. The
existing bytes are shifted left by `a2` bytes, and the queried bytes must be
written at the end of the key.

The variable pre-image key offset is reset to `0` everytime in this case, leaving a
place for optimisation in the MIPS code by avoiding calling `8` times the syscall
if keys are not far away.

<!--
The pre-image key offset is used by the syscall `sysRead` on the file descriptor
`fdPreimageRead` is used to know the chunk it is processing.
-->

#### Default case

<!-- DONE -->

In the case of any other value for the output channel, the register `v0` must be
set to `0xFFffFFff` and the error code `0x9` must be written in the `a3`
register.

### `sysFcntl`

<!-- DONE -->

This syscall is used to perform a command on a file descriptor.
The file descriptor to operate on is in the register `a0` and the command is in
`a1`.

Only one command is supported, aliased `F_GETFL`, which gets the flags of the
file descriptor. The command value is `3`. For any other command value, the
register `v0` is set to `0xFFffFFff` and the register `a3` is set to the error
code `0x16` which means the command has not been recognized by the kernel.

In the case of `F_GETFL`, it sets `v0` to `0`, which represents the flag
`O_READONLY`, when the file
descriptor is the standard input (`0`), the pre-image read (`5`) or the hint
read (`3`). It sets `v0` to `1`, which represents the flag `O_WRONLY`, when the file
descriptor is the standard output (`1`), the standard error (`2`), the hint
write (`4`) or the pre-image write (`6`). The register `a3` is set to `0`.


In the case of any other file descriptor value, the register `v0` is set to
`0xFFffFFff` and the register `a3` is set to the error code `0x9`.

## Test plan and functional requirements

The plan is to implement the instruction syscall in the current MIPS zkVM code
and replicate the behavior described above.
Unit tests should be written and should attest all cases above have been supported.

Tests for the error cases must also be supported. For instance, for the syscall
`sysFcntl`, there must be a test verifying that the correct error code is
populated into the register `a3` in the case of an unsupported command.
Randomized inputs could be used to handle a large set of scenarii.

The following MIPS test codes must pass:
- [`oracle_unaligned_write.asm`](https://github.com/ethereum-optimism/optimism/blob/a8a632901242c373f358cd1aa41376405db88bbc/cannon/mipsevm/open_mips_tests/test/oracle_unaligned_write.asm)
[`oracle_unaligned_read.asm`](https://github.com/ethereum-optimism/optimism/blob/18938993d6ad40d4b37fe439e011d91455531e40/cannon/mipsevm/open_mips_tests/test/oracle_unaligned_read.asm)
- [`oracle.asm`](https://github.com/ethereum-optimism/optimism/blob/a2cc2f41fc7cf270b3e44471a7a4e4931ccaf121/cannon/mipsevm/open_mips_tests/test/oracle.asm)
- [`mmap.asm`](https://github.com/ethereum-optimism/optimism/blob/68cf67fd76d21bc6a28806ca05804a1ab907de91/cannon/mipsevm/open_mips_tests/test/mmap.asm)
- [`fcntl.asm`](https://github.com/ethereum-optimism/optimism/blob/68cf67fd76d21bc6a28806ca05804a1ab907de91/cannon/mipsevm/open_mips_tests/test/fcntl.asm)
- [`exit_group.asm`](https://github.com/ethereum-optimism/optimism/blob/68cf67fd76d21bc6a28806ca05804a1ab907de91/cannon/mipsevm/open_mips_tests/test/exit_group.asm)
- [`brk.asm`](https://github.com/ethereum-optimism/optimism/blob/68cf67fd76d21bc6a28806ca05804a1ab907de91/cannon/mipsevm/open_mips_tests/test/brk.asm)

For IO operations, we must check that at most 4 bytes are read, i.e. the
register `v0` is at most `4`. This can be tested by randomizing the IO syscalls
and check the output is always at most `4`.

## Drawbacks

N/A

## Rationale and alternatives

This design follows the one implemented in the reference VM, Cannon. No other
design has been thought as the zkVM must mimick the Cannon interpreter.

## Prior art

Cannon implements the different syscalls. No known implementations of zkVM implementing OP MIPS syscalls.

## Unresolved questions

N/A

## Resources

For the constants regarding the standard input values: https://github.com/ethereum-optimism/optimism/blob/2251398b856b98ffddcf323daa4bb019f84fc81d/cannon/mipsevm/instrumented.go#L32

For the different syscall values that must be handled:
https://github.com/ethereum-optimism/optimism/blob/2251398b856b98ffddcf323daa4bb019f84fc81d/cannon/mipsevm/mips.go#L45.
It also contains the different registers that must be written and read.

Cannon VM: https://github.com/ethereum-optimism/optimism/blob/develop/specs/cannon-fault-proof-vm.md

## Note

The code must be written in
https://github.com/o1-labs/mips-demo/blob/demo/mips/kimchi/src/mips/instructions/mod.rs#L494
