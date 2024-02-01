# Specialised zkCPU and message passing SNARK argument

## Introduction

This document will describe the construction of a technique coined specialised
zkCPU and will formalized a message-passing protocol between the machines.
Specialised zkCPU can be used to implement what the industry calls zkVM.

## RAMLookup

TODO: see HackMD

## MIPS zkCPU

## Keccak zkCPU

## Passing messages between zkCPU

RAMLookup can be used to register communications between different zkCPU. The
communication happens on a shared tape. The zkCPU will read into a shared tape
(represented as an accumulator in RAMLookup) and the other one will write into
it.

Let's say machine $M1$ records a communication with message $M_{2}$ on its tape
at step $i$. As described in the section RAMLookup, the machine $M_{1}$ will
write the message into the shared tape, adding a random folding of the message
to the accumulator representing the shared tape, in addition to keeping the
components of the messages in its state. Using a PIOP, it can be represented as
keeping the component of the messages as evaluations (in a "column") of a
polynomial the prover will commit to.

The machine $M_{2}$ will read the data by substracting the value to the shared
tape, and by keeping track of the shared tape access in its state, like the
machine $M_{1}$ did.

The execution of both specialised zkCPU can happen in parallel, and the
verification of both executions can be done succintly, in addition to the
verification of the messages shared by the machines.

### Example between MIPS and Keccak zkCPU

## Open questions

How can we represent a communication channel where more than one messages are
written by $M_{1}$ and later read by $M_{2}$?
At the moment, we could simply use different shared tapes. The shared tapes
would be "one-time" use.

## Implementation

See o1VM.
