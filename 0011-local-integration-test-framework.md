# Local Integration Extension to Lucy

## Summary

[Lucy](https://github.com/MinaProtocol/mina/blob/compatible/src/app/test_executive/README.md) is an end-to-end integration testing framework for the Mina protocol. It is designed with the goal of being able to test the protocol with configurable network topologies and node configurations.

This RFC proposes an extension to Lucy to allow for local integration testing. This will allow for easier development of the protocol as well as allow for contributors to run integration tests on their local machines without having to set up a cloud environment.

## Motivation

Lucy has an extensible architecture that allows for the addition of different networking and deployment backends. Currently, Lucy runs off a Kubernetes cloud backend (specifically, GKE is used), where it uses Terraform to deploy a Kubernetes cluster and Helm to deploy the network and specified nodes. While this method works for cloud environments, we would like a more lightweight solution to run locally. Thus, we would like to have a local backend that can be used to run the integration tests on a local machine.

Instead of using a Kubernetes backend to run the integration tests in the cloud, we can use containers to run and deploy the network configuration on a local machine. The aim is that these containers will be lighter-weight than a full Kubernetes cluster and will allow for easier development of the protocol.

## Detailed Design

### Functional Requirements

To implement a local backend for Lucy, we will use Docker containers to run the network configuration on a local machine. Docker containers are a lightweight way to run applications in an isolated environment. They are a great way to run applications in a consistent environment and are a great way to run applications locally. By using Docker containers, we can run the network configuration on a local machine without having to worry about the specifics of running the network on a local machine.

We want the following properties for the local backend:

- **Lightweight**: The local backend should be lightweight and should not require a lot of resources to run. This is important as we want to be able to run the network on a local machine without having to worry about the resource requirements of the network.
- **Easy to use**: The local backend should be easy to use and should not require a lot of setup to run. This is important as we want to be able to run the network on a local machine without having to worry about the network setup.
- **Fast**: The local backend should be fast and should not take a long time to run. This is important as we want to be able to run the network on a local machine without having to worry about the time it takes to run the network.

### Orchestration

To run containers locally, we will use [Docker Swarm](https://docs.docker.com/engine/swarm/) as the main driver. Docker Swarm is a container orchestration tool that allows us to run containers on a local machine or on a cluster of machines. It's built into the Docker Engine and there's no need for a seperate installation using it. The main reason to use this over [Docker Compose](https://docs.docker.com/compose/), is that Docker Swarm has multi-host networking built in, so if we want to further extend Lucy to run on a cluster of machines with Docker, we can do so easily. Docker Swarm takes a Docker Swarm file as input, which specifies all the containers to run and their configurations. To create a correct Docker Swarm file for our integration tests, we can create a mapping of the network configuration used to specify the tests and a Docker Swarm file. The network configuration is defined at the beginning of each integration test, which is then used to create a corresponding docker swarm file that has the specified network topology to use. We can use this file to specify the network topology configuration and then run the network on a local machine.

Using Docker Swarm allows us to handle the orchestration of containers without having to worry about the specifics of running containers on a local machine. Docker Swarm will handle all the networking, log collection, and resource management for us.

### Extending Lucy

Lucy is designed with an extensible architecture that allows for the addition of different networking and deployment backends. Lucy extends an [OCaml module interface (named `Engine`)](https://github.com/MinaProtocol/mina/blob/compatible/src/lib/integration_test_lib/intf.ml), which offers an abstract way to specify different testing engines when running the test executive. It defines several important behaviours that must be implemented by any backend engine that is used. This module interface is the contract between the backend engine and how Lucy expects the network to be deployed and managed. The `Engine` module interface defines the following:

- [`Network_config_intf`](https://github.com/MinaProtocol/mina/blob/c396e82af6da7f69817b8885e46b4da94715e27e/src/lib/integration_test_lib/intf.ml#L20):
  - Takes as input a user-defined network topology configuration and generates a corresponding file that specifies how the specified backend engine should deploy the network. In the case of the current Kubernetes backend, it generates a Terraform file that specifies the Kubernetes cluster configuration. For the local backend, it will generate a docker swarm file that specifies the network topology configuration.
- [`Network_intf`](https://github.com/MinaProtocol/mina/blob/c396e82af6da7f69817b8885e46b4da94715e27e/src/lib/integration_test_lib/intf.ml#L39):
  - Specifies how to interact with the network that is deployed. Several behaviours include starting/stopping nodes, checking the status of nodes, checking the network information itself, etc. The Kubernetes backend uses the Kubernetes API to interact with the network that is deployed by issuing commands like `kubectl exec ...` for example. The local backend will operate in the same way but instead use the Docker API to interact with the network that is deployed (e.g. `docker exec ...`).
- [`Network_manager_intf`](https://github.com/MinaProtocol/mina/blob/c396e82af6da7f69817b8885e46b4da94715e27e/src/lib/integration_test_lib/intf.ml#L99):
  - Specifies how to deploy the network. Functions like `create,` `destroy,` `deploy,` and `clean up` are all defined in this interface. For the Kubernetes backend, this means deploying a Kubernetes cluster to GKE and initializing the network. For the local backend, this means running the docker swarm file and initializing the network.
- [`Log_engine_intf`](https://github.com/MinaProtocol/mina/blob/c396e82af6da7f69817b8885e46b4da94715e27e/src/lib/integration_test_lib/intf.ml#L115):
  - Specifies how to gather logs from the network. This is important as this is how Lucy gathers network information and confirms if integration test conditions pass or fail. The Kubernetes backend uses the Mina GraphQL API to poll for network state and stores those logs in Google Stack Driver. The local backend will use a pipe to gather logs from the network in a similar fashion.

As a comparison, Kubernetes specifies a [config type](https://github.com/MinaProtocol/mina/blob/7d11e9bb3a45cfdda8d06d1d4819ef3a31dd2983/src/lib/integration_test_cloud_engine/mina_automation.ml#L57) that is used to specify the Kubernetes cluster configuration. This config type is then used to generate a Terraform file that specifies the topology and configuration of the Kubernetes cluster. This Terraform file is then used to deploy the Kubernetes cluster to GKE. We will do the same thing for the local backend, but instead of generating a Terraform file, we will generate a Docker Swarm file that specifies the topology and configuration of the network. This Docker Swarm file will then be used to deploy the network on a local machine. One aspect of the Docker Swarm file to highlight is that it will be in JSON format, so no new libraries to handle YAML will be necessary. The configuration type could look something like this:

```ocaml
type docker_config =
    { docker_swarm_version : string
    ; stack_name : string
    ; mina_image : string
    ; mina_agent_image : string
    ; mina_bots_image : string
    ; mina_points_image : string
    ; mina_archive_image : string
    ; runtime_config : Yojson.Safe.t
    ; docker_volume_configs : docker_volume_configs list
    ; seed_configs : seed_config list
    ; block_producer_configs : block_producer_config list
    ; snark_coordinator_config : snark_coordinator_config option
    ; archive_node_configs : archive_node_configs list
    ; log_precomputed_blocks : bool
    ; cpu_request : int
    ; mem_request : string
    ; worker_cpu_request : int
    ; worker_mem_request : string
    }
  [@@deriving to_yojson]
```

This type is fed into the `Engine` interface, which then specifies how to transform this type into a Docker Swarm file and deploys it to a local machine.

### Node Communication/Logging

_Note: The following is out-dated but kept for historical reasons._ To see up to date details on how node communication and logging will work, see [here](https://github.com/MinaProtocol/mina/pull/14648).

Node logging and information gathering is done by [GraphQL queries](https://github.com/MinaProtocol/mina/blob/compatible/src/lib/integration_test_lib/graphql_requests.ml) to the Mina daemon. These queries are what allow Lucy to gather information about the network and confirm if integration test conditions pass or fail. When the Kubernetes backend is deployed, nodes are deployed via the user-defined network configuration. They are then started and [polled for logs](https://github.com/MinaProtocol/mina/blob/78535ae3a73e0e90c5f66155365a934a15535779/src/lib/integration_test_cloud_engine/graphql_polling_log_engine.ml#L122) and network information. This polling approach works for the Kubernetes backend as the logs are pre-filtered (so the volume of logs is very low), and the logs parsing is fast enough in this case. However, the polling approach can be a performance bottleneck for the local backend as the volume of logs can be very high if we run SNARKless networks. These types of networks run much faster by comparison, where a 5-10 second delay in log gathering could break the running test. Furthermore, the log volume per second of a SNARKless network is massive by comparison due to the speed at which the network operates. For this reason, polling is not the best approach for the local backend.

Instead, the local backend will use a push-based approach, where all logs from nodes are forwarded to Lucy. This approach will be implemented by using a [pipe](https://man7.org/linux/man-pages/man3/mkfifo.3.html) that is shared between all containers in the network. This pipe will be created by Lucy and will be mounted in every container specified in the docker swarm file. This pipe will then act as the communication between all container logs and the Lucy.

Lucy will create a pipe in the current directory and will include that file as a bind mount for each container in the docker swarm file. Furthermore, we can tag all container logs with a unique identifier so that Lucy can filter logs by the container. This will allow us to deal with scenarios where we read duplicate logs and allow for easier filtering.

A further optimization we will apply is using [`logproc`](https://github.com/MinaProtocol/mina/blob/compatible/src/app/logproc/logproc.ml) to all container output before it's written to the pipe. `logproc` can help us filter logs by log level, which will help us reduce the volume of logs that are written to the pipe. This will help us reduce the number of logs that are written to the pipe and the number of logs that Lucy has to parse.

Concretely, we will include an entry point script for each container that redirects stdout and stderr to the named pipe with a prefix of the container name. This will allow us to filter logs by container name and will allow us to deal with duplicate logs. An example of this script is shown below:

```bash
#!/bin/bash
NAMED_PIPE="/path/to/named_pipe"
CONTAINER_ID=${CONTAINER_ID:-"unknown_container"}
prepend_container_id() {
    while IFS= read -r line; do
        echo "$CONTAINER_ID: $line"
    done
}
cat - | prepend_container_id | logproc -i inline -f '!(.level in ["Spam", "Debug"])' | tee -a $NAMED_PIPE
```

Then, inside the docker swarm file, we can specify the entrypoint script to use for each container.

```yaml
version: "3.8"
services:
  node:
    image: minaprotocol/mina-daemon:...
    entrypoint: /bin/bash -c "/path/to/puppeteer-context/start.sh $$@ | /usr/local/bin/log_processor.sh"
    environment:
      - CONTAINER_ID=mina-node-1
    volumes:
      - ./pipe:/path/to/pipe
      - ./log_processor.sh:/usr/local/bin/log_processor.sh
```

## Drawbacks

Because we are running all Mina daemon nodes on a single local machine, we will be limited by the resources of that machine. This means that we will not be able to run large networks on a single machine. However, this is not a problem as we can run smaller networks on a single machine and run larger networks in a cloud environment.

## Rationale and alternatives

By using Docker containers, we can run the network configuration on a local machine without having to worry about the specifics of running the network. Containers are also portable and consistent across all machines, which makes it easy to run the network on any machine (e.g. inside CI).

One could argue that we could leverage the existing Kubernetes infrastructure and use Kubernetes locally instead of targeting the cloud. This is a valid alternative, but the existing Kubernetes backend is tightly coupled towards GKE, which makes it difficult to port over the existing backend to target a local cluster instead. Furthermore, using containers for local network deployment offers a simple mental model for how the network is deployed and managed compared to Kubernetes.

By not implementing this RFC, we will not be able to run integration tests locally. This will make it difficult for contributors to run integration tests locally and will make it difficult to develop the protocol locally.

## Unresolved questions

- What are the limits of the local backend? How many nodes can we run on a single machine? For a given hardware configuration, it's not clear how many nodes we can run on a single machine. This is something that we will have to test and figure out.

- Are there differences in how the chosen tools will operate between operating systems? Could there be performance or stability issues on certain operating systems? This is something that we will have to test and figure out.
