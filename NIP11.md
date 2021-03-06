# NIP 11 - Symbol Configuration Utility (CLI)

## Summary

```
    NIP: 11
    Layer: Application
    Title: Symbol Configuration Utility (CLI)
    Author: Bader Youssef <bader@iodlt.com>
            David Garcia <david.garcia@nem.software>
    Discussions-To: #sig-client
    Comments-URI: https://github.com/nemtech/NIP/issues/35
    Status: Draft
    Type: Standards Track
    Created: 2020-02-03
    License: MIT
```
## Table of contents

- [Introduction](#introduction)
- [Motivation](#motivation)
- [Specification](#specification)
  * [Command layout outline](#command-layout-outline)
    + [network install](#network-install)
    + [network download](#network-download)
    + [network create](#network-create)
    + [node download](#node-download)
    + [node connect](#node-connect)
    + [node changerole](#node-changerole)
    + [node reset](#node-reset)
    + [node stop](#node-stop)
- [Design Decisions](#design-decisions)
- [Implementation](#implementation)
- [References](#references)
- [History](#history)

## Introduction

This NIP defines the specification of a tool to create new chains and connect nodes to existent networks.

## Motivation

The current configuration process of a Symbol blockchain leaves much to be desired.
To configure a Symbol instance, there is either the option of using one of the many pre-configured Docker images, which does not work on all systems (ARM-based devices, older systems), or completely configuring from scratch.

To deploy a Symbol (peer) instance, a user requires to perform the next steps:

1. Download, compile and install catapult-server dependencies.

2. Configure _19_ properties files for a new chain instance and create the nemesis block. For joining an existing chain, the proper properties for that chain (_e.g a testnet_) must be downloaded and placed in the correct path.

3. Configure the node's settings in [`config-node.properties`](https://github.com/nemtech/catapult-server/blob/main/resources/config-node.properties), e.g a philanthropic node setup.

4. Load the `config-extension` files based on node role: `DUAL`, `API`, or `PEER`. For developers, switching and testing different network configurations is often part of the daily workflow.

5. Launch the node.

There is currently no streamlined, easily installable tool to accomplish the above. Attempts were made with [`cat-config-scripts`](https://github.com/IoDLT/cat-config-scripts/) and [`symbol-testnet-bootstrap`](https://github.com/nemgrouplimited/symbol-testnet-bootstrap/) but they do not offer portability. The tool [catapult-service-bootstrap](https://github.com/nemtech/catapult-service-bootstrap) is another great tool to create new networks, but not sufficient to deploy new nodes. 

This NIP solves this problem by offering a CLI utility that streamlines the above into an easy-to-use tool. When implemented and adopted, NIP11 could replace the above-mentioned projects.

## Specification

### Command layout outline

The following is a proposed command tree for the CLI.
Each command should contain a parent command, along with the required parameters.

#### network install*

Download the latest compatible versions of catapult-server and catapult-rest, install the required dependencies, and compiles the server.
The user should be able to define custom endpoints to download the repositories and custom version tags.

USAGE

symbol-bootstrap network install [...options]

OPTIONS

| Option                                | Required | Default                                        | Description                             |
|---------------------------------------|----------|------------------------------------------------|---------------------------------------- |
| ``--server-git <server-git>``         | No       | https://github.com/nemtech/catapult-server.git | Link to catapult-server.                |
| ``--server-version <server-version>`` | No       | e.g. v1.0.0                                    | catapult-server branch or tag.          |
| ``--rest <rest>``                     | No       | https://github.com/nemtech/catapult-rest.git   | Link to catapult-rest.                  |
| ``--rest-version <rest-version>``     | No       | e.g. v1.0.0                                    | catapult-server branch or tag.          |

#### network download

Downloads the recommended network settings, including the nemesis block configurable definition, and peers files from a given url.

`*` Build a separate service to share the default download URLs with randomized values for ``networkGenerationHashSeed``, ``networkMosaicIds``, ``distribution`` list, and ``sinkAddresses``. As a future enhancement, provide a multi-step form to define the nemesis block properties file step by step.

USAGE

symbol-bootstrap network download [...options]

OPTIONS

| Option                              | Required | Default                                | Description                         |
|-------------------------------------|----------|----------------------------------------|-------------------------------------|
| ``--resources <resources>``         | Yes      | http://default.url/resources           | URL  compressed folder.             |
| ``--nemesis-properties <nemesis-properties>``  | No |                                   | URL to nemesis config file.         |
| ``--output <output>``               | No       |./resources                             | Destination folder.                 |

#### network create

Creates the nemesis block inside the ``data`` folder.
Before running the command, configure the nemesis block and [network properties](https://nemtech.github.io/guides/network/configuring-network-properties.html) files.

*Note*: The nemesis block links all the eligible harvesters with random VRF and Voting Keys.
These keys should be returned to the user if the nemesis block is created successfully.

USAGE

symbol-bootstrap network create [...options]

OPTIONS

| Option                              | Required | Default                                | Description                  |
|-------------------------------------|----------|----------------------------------------|------------------------------|
| ``--server <server>``               | No       | ./catapult-server/bin                  | Path to compiled server.     |
| ``--resources <resources>``         | No       | ./resources                            | Path to resources folder.    |
| ``--nemesis-properties <nemesis-properties>``  | No | ./resources/mijin-test.properties | Path to nemesis config file. |
| ``--output <output>``               | No       | ./template                             | Path where ``data``  and ``seed`` folders will be created.  |
| ``--docker``                        | No       |                                        | If set, ``server`` accepts a docker image instead of a path.|

#### node download

Downloads the template files required to start and join a Symbol instance.

The notion of network ``templates`` exists within [`cat-config-scripts`](https://github.com/IoDLT/cat-config-scripts/).
A user should be able to download the resources and nemesis for a specific network as a "template", and specify the path to this template in the configuration file, or via the CLI. A template should follow the following file structure:

```
example-network-template/
.
????????? seed
????????? data
????????? resources
```

Where `data/` contains the active directory for the node to store new chain information, `seed/` contains original nemesis data in the case of a node reset, and `resources` contains network-specific files. Users will be able to download network configurations in this format and load them via the CLI.

USAGE

symbol-bootstrap node download [...options]

OPTIONS

| Option                              | Required | Default                           | Description                           |
|-------------------------------------|----------|-----------------------------------|---------------------------------------|
| ``--template <template>``           | Yes      |                                   | URL to template compressed folder.    |
| ``--output <output>``               | No       | ./template                        | Destination folder.                   |


#### node connect

Launches a Symbol node and connects it to an existent network.
Before running the command, the user should configure the node properties.

USAGE

symbol-bootstrap node connect [...options]

OPTIONS

| Option                              | Required | Default                           | Description               |
|-------------------------------------|----------|-----------------------------------|---------------------------|
| ``--server <server>``               | No       | ./catapult-server/bin             | Path to compiled server.  |
| ``--rest <rest>``                   | No       | ./catapult-rest/                  | Path to rest with dependencies installed. Required if API or Dual Node. |
| ``--resources <resources>``         | No       | ./template/resources              | Path to the resources folder.                    |
| ``--data <data>``                   | No       | ./tempalte/data                   | Path to the data folder.                         |
| ``--seed <seed>``                   | No       | ./template/seed                   | Path to the seed folder.                         |
| ``--certificate <certificate>``     | No       |                                   | Path to the certificates folder. If not set, new self-signed certificates are generated. |
| ``--detach``                        | No       |                                   | Launches the node in the background.             |
| ``--docker``                        | No       |                                   | If set, ``server`` and ``rest`` accepts a docker image instead of a path.|

#### node changerole*

Updates the node role by overwriting the resources configuration files.

USAGE

symbol-bootstrap node changerole [...options]

OPTIONS

| Option                              | Required | Default                           | Description                                                        |
|-------------------------------------|----------|-----------------------------------|--------------------------------------------------------------------|
| ``--resources <resources>``         | No       | ./template/resources              | Path to the resources folder.                                      |
| ``--role <role>``                   | Yes      | peer                              | Overwrites the template configuration. Options: peer, api or dual. |

#### node reset

Resets the data from a local node.

USAGE

symbol-bootstrap node reset

#### node stop

Stops the node.

USAGE

symbol-bootstrap node stop

*Note*: If nodes are not using containers, only one node can be run at a time to avoid having conflicts with ports. If the computer is running a node already, this should be stopped first before running ``connect``. `*` Otherwise, the ``reset`` and ``stop`` commands will have an extra option to pass the node's unique id and an additional command to list all active nodes.

`*` Will most likely not be implemented in the first version(s) of the CLI.	

## Design Decisions

- The CLI application makes the most sense for a cross-platform chain configuration tool. GUI applications cannot always be used in remote server configuration, and bash scripts can be cumbersome.

- The Symbol Configuration Utility will be written in Typescript and use the [Command Design Pattern](https://en.wikipedia.org/wiki/Command_pattern). This pattern suits CLI-style applications very well, as it allows for the command logic to be encapsulated for later use depending on the program's state.

- The NIP11 will be published as an NPM package, and thus installation is the same as any NPM global module. It's possible that this implementation could be integrated into the existing [`symbol-cli`](https://github.com/nemtech/symbol-cli), which also happens to use the same software design pattern as proposed above. In this case, the functionality would be implemented as part of new commands.

## Implementation

There is no implementation yet.

## References

- [catapult-server](https://github.com/nemtech/catapult-server)
- [catapult-rest](https://github.com/nemtech/catapult-rest)
- [cat-config-scripts](https://github.com/IoDLT/cat-config-scripts/)
- [nem2-cli](https://github.com/nemtech/nem2-cli)
- [catapult-service-bootstrap](https://github.com/nemtech/catapult-service-bootstrap)
- [symbol-testnet-bootstrap](https://github.com/nemgrouplimited/symbol-service-bootstrap)

## History

| **Date**     | **Version**    |
| -------------| -------------  |
| Feb 3 2020   | Initial Draft  |
| Jul 7 2020   | Second Draft   |
| Jul 27 2020  | First Revision |

