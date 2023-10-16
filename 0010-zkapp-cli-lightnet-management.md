# RFC for zkapp-cli enhancements to provide the local lightnet networks management

Introduce new sub-commands to the existing zkapp CLI application to streamline the local lightnet networks management for development and testing purposes.

## Summary

We aim to extend the current zkapp CLI application by introducing the `lightnet` sub-command suite. This suite will centralize and simplify Docker containers and provided tools management tasks for local zkApps development and testing, enhancing user and developer experience.

## Motivation

The primary motivation is to simplify the local zkApps development and testing process by offering intuitive CLI commands.

Expected Outcomes:

- A streamlined workflow for managing local lightnet network [Docker containers](https://hub.docker.com/r/o1labs/mina-local-network) and tools provided by such containers.
- Reduced complexity and better DX, eliminating manual 3rd party tools management.

## Detailed design

### zkapp CLI Interface Enhancements

Under the primary command `zk`, the new `lightnet` sub-commands will be introduced:

#### Help

- **Command**: `zk lightnet`
- **Output**: Help message with description of available sub-commands and their options.

#### Start lightnet network

- **Command**: `zk lightnet start <options>`
- **Function**: Checks the env., starts the local lightnet Docker container with reasonable default configuration options and waits until the network sync. Repeating the command while network is up and running should not start a new container, but rather check the status of the existing one.
- **Configuration options**:
  - --mode: `single-node` or `multi-node` (default: `single-node`)
  - --proof-level: `none` or `full` (default: `none`)
  - --archive-node: `true` or `false` (default: `true`)
  - --mina-branch: `rampup`, `berkeley` or `develop` (default: `rampup`)
  - --dune-profile: `lightnet` or `devnet` (default: `lightnet`)
- **Output**:
  - Anticipated network properties (as per [Docker image description](https://hub.docker.com/r/o1labs/mina-local-network));
  - Mina Daemon GraphQL endpoint for communication: [http://localhost:8080/graphql](http://localhost:8080/graphql)
  - PostgreSQL connection string: `postgresql://postgres:postgres@localhost:5432/archive`;
  - Mina Daemon and Archive Node logs path;
  - [o1js configuration](https://github.com/o1-labs/o1js/blob/ccf50e0b58190d9700c6dda4fafee3e62e270131/src/examples/zkapps/hello_world/run_live.ts#L18) code snippet.

#### Stop lightnet network

- **Command**: `zk lightnet stop`
  - In future we might want to extend this command with additional options to preserve the network state between containers lifecycle.
- **Function**: Destroys the running container. Repeating the command while network is down should not throw an error but rather output a message that the network is already down.
- **Output**: Maybe blockchain state (blocks height, slots, epoch, txns processed during the session, etc.) at the time of command issuing.

#### Handle logs

- **Command**: `zk lightnet logs <sub-command>`
- **Sub-commands**:
  - **view**: `less` or `tail` the log files (Mina Daemon or Archive Node). OS agnostic.
  - **save**: Saves the log files to default location and outputs the resulting path.

#### Manage accounts

- **Command**: `zk lightnet account <sub-command>`
- **Sub-commands**:
  - **fetch**: Fetches available account from the container's pool.
  - **release**: Releases "used" account back to the pool.
- **Note**: Corresponding [o1js API](https://github.com/o1-labs/o1js/pull/1167) can be used.

#### Launch the blockchain explorer

- **Command**: `zk lightnet explorer`
- **Function**: Opens the local lightweight blockchain explorer (HTML + JS + CSS application) in default browser tab.
  - Existing lightweight explorer will be refactored and ported under the [zkapp-cli](https://github.com/o1-labs/zkapp-cli) repository.
- **Output**: The URL of application.

### Impact

This update enhances the existing zkapp CLI by appending more commands under the `lightnet` sub-command. The core operations will remain intact.

## Test plan and functional requirements

- **Testing goals and objectives**:
  - To validate the reliability of the enhanced zkapp CLI tool.
- **Testing approach**:
  - Unit tests for the new `lightnet` subcommands.
  - Integration tests for the new `lightnet` subcommands.
- **Testing scope**:
  - Testing all lightnet commands and their possible variations.
  - Addressing potential error scenarios and recovery mechanisms.
- **Testing requirements**:
  - Test in various local setups to simulate different development environments.
- **Testing resources**:
  - Simulated environments resembling typical local development setups.

## Drawbacks

Don't see any.

## Rationale and alternatives

Not enhancing the zkapp CLI would force developers to use intricate Docker commands and additional tools manually, impacting DX.
