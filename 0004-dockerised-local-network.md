# Dockerization of Mina Local Networks

## Summary

This RFC proposes the creation of scripts required to produce a Docker image for Mina Local Networks, providing a simple and efficient way to deploy and run lightweight Mina blockchain networks for testing purposes. The Docker image will encapsulate the Mina daemon and other necessary components, allowing users to quickly spin up a local Mina network with a single command.

## Motivation

The Mina protocol, while powerful and innovative, currently lacks a simple and efficient way to deploy and run local networks for testing purposes. This has led to challenges in both development and user experience (DX/UX), as developers have to either use costly cloud solutions or set up everything locally, both of which can be slow and resource-intensive.

The Dockerization of lightweight Mina Local Networks aims to address these challenges by providing a lightweight, fast, and resource-efficient solution for spinning up local Mina networks. This will significantly improve the DX/UX, allowing developers to easily test their applications and changes against a "close to real" Mina network.

## Detailed design

The Dockerization process involves two main phases:

1. **Building Phase**: Usual build procedure the engineers familiar with. In this phase, the Mina Daemon is built locally or on a CI server using the desired branch, revision, and compile-time constants. The build process can leverage the `lightnet` Dune profile to create a lightweight version of the Mina Daemon.
2. **Dockerization Phase**: In this phase, scripts are invoked to prepare the filesystem (binaries, additional scripts, genesis ledger configuration with pre-funded accounts, etc.) and ensure that the Mina Local Network scripts work well inside the Docker environment. The Docker image is then built and uploaded to Docker Hub using provided credentials (Users have to be authenticated using `docker login` CLI).

The resulting Docker image allows users to spin up a Mina Local Network using a simple command like

```shell
docker run -it --env PROOF_LEVEL=none -p 4001:4001 -p 4006:4006 -p 5001:5001 -p 6001:6001 mina-local-network
```

The Docker image can also be used in CI jobs to run end-to-end tests, particularly for tools that don't require fully fledged networks but still need to test integration with "close to real" networks. This significantly improves the developer experience (DX) and user experience (UX). Example candidates for this kind of testing include tools like [SnarkyJS](https://github.com/o1-labs/snarkyjs), [zkapp-cli](https://github.com/o1-labs/zkapp-cli), and [Archive-Node-API](https://github.com/o1-labs/Archive-Node-API).

The lightweight Mina Local Network created by the Docker image will have the following properties:

- Transaction finality (`k`) is `30` blocks.
- There are `720 slots` per `epoch`.
- New blocks are produced every `~20-40` seconds.
- Each block contains `5-8` transactions.
- The network consumes `~4.5GB` of RAM at the beginning on `Linux` platform and `~5.5Gb` of RAM on `macOS`.
- The startup and sync time is approximately `4` minutes.

For more details on how to manually set up and use the lightweight Mina Network, please refer to the [Mina Local Network Manager](https://github.com/MinaProtocol/mina/tree/rampup/scripts/mina-local-network#mina-lightweight-network) documentation on GitHub.

## Test plan and functional requirements

1. **Testing goals and objectives**: The primary goal is to ensure that the Docker image can successfully spin up a Mina Local Network and that this network behaves as expected. This includes fast startup and syncing times, low resource consumption, and the ability to form a network with diverse types of participants.
2. **Testing approach**: The testing will involve the manual functional testing only, at least for now, because of the low scripts complexity. Performance monitoring will also be conducted to ensure that the Docker image meets the resource efficiency requirements.
3. **Testing scope**: The testing will cover all major functionalities of the Docker image, including Dockerization phase, and the resulting Mina Local Network operation.
4. **Testing requirements**: The testing will require a suitable environment for running Docker, as well as tools for monitoring resource usage and blockchain behavior.
5. **Testing resources**: The testing will be conducted on a local machine. The testing may also require additional tools for monitoring and debugging.

## Drawbacks

One drawback of this approach is the potential for discrepancies between the behavior of the lightweight Mina Local Network and the full Mina network. While the lightweight network is designed to be "close to real", it may not perfectly replicate all aspects of the full network, which could lead to misleading test results. To mitigate this drawback, we can state clearly in the documentation that the lightweight network is not intended to be a perfect replica of the full network and that users/developers should always double check their work against real networks.

Another potential drawback is the need for Docker as a dependency. Users who do not have Docker installed or who are unfamiliar with Docker may face additional hurdles in setting up and using the Mina Local Network.

## Rationale and alternatives

The Dockerization of Mina Local Networks is proposed as a solution to the current challenges in deploying and running local Mina networks for testing purposes. By encapsulating the Mina daemon and other necessary components in a Docker image, we can provide a simple, fast, and resource-efficient solution for spinning up local Mina networks.

An alternative to this approach would be to improve the existing methods for deploying and running Mina networks, such as optimizing the setup process or providing better support for cloud-based solutions. However, these alternatives would likely be more complex and time-consuming to implement and would not offer the same level of simplicity and efficiency as the Docker-based solution.

## Prior art

Dockerization is a common practice in software development, particularly for applications that require complex setup processes or that need to run in isolated environments. Many blockchain projects also provide Docker images for their nodes or other components, recognizing the benefits of Docker in terms of simplicity, reproducibility, and resource efficiency.

## Unresolved questions

- What is the best way to handle the building phase? And how can we ensure that the Docker image is kept up-to-date with the latest changes to the Mina Daemon and other components?
  - Should we add additional Minaprotocol CI job to periodically (say, once a day) build the lightweight Mina Daemon, build Docker image and publish it on behalf of O(1) Labs Docker Hub account?
  - Which Mina branches should be used to prepare the corresponding Docker images with the lightweight Mina Network inside?
- How can we best handle the potential discrepancies between the lightweight Mina Local Network and the full Mina network?