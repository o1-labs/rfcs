# Dockerization of Mina local networks

## Summary

This RFC proposes the creation of scripts required to produce a Docker image for Mina local networks, providing a simple and efficient way to deploy and run lightweight Mina blockchain networks for testing purposes. The Docker image will encapsulate the Mina daemon and other necessary components, allowing users to quickly spin up a local Mina network with a single command.

## Motivation

The Mina protocol, while powerful and innovative, currently lacks a simple and efficient way to deploy and run local networks for testing purposes. This has led to challenges in both development and user experience (DX/UX), as developers have to either use costly cloud solutions or set up everything locally, both of which can be slow and resource-intensive.

The Dockerization of lightweight Mina local networks aims to address these challenges by providing a lightweight, fast, and resource-efficient solution for spinning up local Mina networks. This will significantly improve the DX/UX, allowing developers to easily test their applications and changes against a "close to real" Mina network.

## Detailed design

The Dockerization process involves two main phases:

1. **Building Phase**: Usual build procedure the engineers familiar with. In this phase, the Mina Daemon is built locally or on a CI server using the desired branch, revision, and compile-time constants. The build process can leverage the [lightnet](https://github.com/MinaProtocol/mina/blob/4e0b324912017c3ff576704ee397ade3d9bda412/src/config/lightnet.mlh) Dune profile to create a lightweight version of the Mina Daemon.
2. **Dockerization Phase**: In this phase, scripts are invoked to prepare the file system (binaries, additional scripts, genesis ledger configuration with pre-funded accounts, etc.) and ensure that the Mina local network scripts work well inside the Docker environment. The Docker image is then built and uploaded to preferred cloud registry (for example `GCP Artifact Registry`, `Docker Hub`, `AWS Elastic Container Registry`).

The resulting Docker image allows users to spin up a Mina local network using a simple command like

```shell
docker run -it --env NETWORK_TYPE="single-node" --env PROOF_LEVEL="none" -p 3085:3085 -p 8080:8080 -p 8181:8181 <org>/mina-local-network:<tag>
```

The Docker image can also be used in CI jobs to run end-to-end tests, particularly for tools that don't require fully fledged networks but still need to test integration with "close to real" networks. This significantly improves the developer experience (DX) and user experience (UX). Example candidates for this kind of testing include tools like [SnarkyJS](https://github.com/o1-labs/snarkyjs), [zkapp-cli](https://github.com/o1-labs/zkapp-cli) and [Archive-Node-API](https://github.com/o1-labs/Archive-Node-API), as well as the 3rd party applications built using these tools.

### Solution options and features

- It will allow users to choose between a `single-node` network and a `multi-node` network. Either to run a single node or a network with diverse types of participants.
- It will provide a `Genesis Ledger` with pre-funded accounts, allowing users to quickly start testing their applications.
- It will provide the `Accounts-Manger` service which will allow users to retrieve accounts to work with in automated way.
- It will provide an additional port served by reverse proxy and passing requests to the Mina Daemon's `GraphQL` endpoint in order to properly manage the `CORS` configuration so that the `SnarkyJS` (formerly known as `SnarkyJS`) applications can work with the network without any additional configuration.

### Resulting network properties

- Transaction finality (`k`) in `30` blocks.
- `720 slots` per `epoch`.
- New blocks production every `~20-40` seconds.
- `5-8` transactions per block.
- `~625-650Mb` of RAM consumption (after initial spike and if stays alive during less than `~2hrs` = `1/2 epoch`).
- The startup and sync time `~1-2` minutes.

For more details on how to manually set up and use the lightweight Mina Network, please refer to the [Mina local network manager](https://github.com/MinaProtocol/mina/tree/rampup/scripts/mina-local-network#mina-lightweight-network) documentation on GitHub.

## Delivery plan and cadence

The resulting Docker images will be published to the `GCP Artifact Registry`. Accompanying this, we envision updates to our publicly accessible documentation such as [docs.minaprotocol.com](https://docs.minaprotocol.com/) and the relevant GitHub repositories README.md files.  
Our approach involves distribution of Docker images for key branches, like `rampup`, `berkeley`, `develop`.  
To maintain the utility of this solution, regular reviews of user needs and expectations may be conducted. This could lead to the introduction of new features, relevant changes, and adjustments to the focus on particular branches and the frequency of updates.  
The delivery cadence for the Docker images is currently set to be daily.

## Test plan and functional requirements

1. **Testing goals and objectives**: The primary goal is to ensure that the Docker image can successfully spin up a Mina local network and that this network behaves as expected. This includes fast startup and syncing times, low resource consumption, and the ability to form a network with diverse types of participants.
2. **Testing approach**: The testing will involve the manual functional testing only, at least for now, because of the low scripts complexity. Performance monitoring will also be conducted to ensure that the Docker image meets the resource efficiency requirements.
3. **Testing scope**: The testing will cover all major functionalities of the Docker image, including Dockerization phase, and the resulting Mina local network operation.
4. **Testing requirements**: The testing will require a suitable environment for running Docker, as well as tools for monitoring resource usage and blockchain behavior.
5. **Testing resources**: The testing will be conducted on a local machine. The testing may also require additional tools for monitoring and debugging.

## Drawbacks

One drawback of this approach is the potential for discrepancies between the behavior of the lightweight Mina local network and the full Mina network. While the lightweight network is designed to be "close to real", it may not perfectly replicate all aspects of the full network, which could lead to misleading test results. To mitigate this drawback, we can state clearly in the documentation that the lightweight network is for local development only and is not intended to be a perfect replica of the full network. Mina local networks do not guarantee production-ready results. Users and developers must always double-check their work against real networks.

Another potential drawback is the need for Docker as a dependency. Users who do not have Docker installed or who are unfamiliar with Docker may face additional hurdles in setting up and using the Mina local network.

## Rationale and alternatives

The Dockerization of Mina local networks is proposed as a solution to the current challenges in deploying and running local Mina networks for testing purposes. By encapsulating the Mina daemon and other necessary components in a Docker image, we can provide a simple, fast, and resource-efficient solution for spinning up local Mina networks.

An alternative to this approach would be to improve the existing methods for deploying and running Mina networks, such as optimizing the setup process or providing better support for cloud-based solutions. However, these alternatives would likely be more complex and time-consuming to implement and would not offer the same level of simplicity and efficiency as the Docker-based solution.

Another future option might be in utilizing [OpenMina](https://openmina.com/) project, which can target `Web` / `JS` runtime and provide a lightweight Mina local network in a browser. This would be a great option for developers, however, it is not available at the moment.

## Prior art

Dockerization is a common practice in software development, particularly for applications that require complex setup processes or that need to run in isolated environments. Many blockchain projects also provide Docker images for their nodes or other components, recognizing the benefits of Docker in terms of simplicity, reproducibility, and resource efficiency.
