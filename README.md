> [!WARNING]
> This repository is a fork of Topology original ts-drp, and it's in the middle of being restructured. Use it with caution.

# Overview

This is the official TypeScript implementation of Distributed Real-Time Programs (DRP). DRP is a local-first decentralized protocol for real-time applications. It is built on top of libp2p and with a similar design with CRDTs.

# Specifications

The specifications of DRP are shared across different client implementations and can be found in the [specs repository](https://github.com/drp-tech/specs). Currently the specifications are starting to be written based on this implementation.

# Packages

This repository is a monorepo that contains the following packages:

| Package    | Description                                     |
| ---------- | ----------------------------------------------- |
| blueprints | Blueprints of some DRPs that can be freely used |
| logger     | Logger for the whole project                    |
| network    | Network middleware to abstract libp2p           |
| node       | Node for interacting with DRPs library and CLI  |
| object     | DRP objects structure implementation            |

# Examples

All the examples are located in the `examples` directory.

# Usage

This workspace has all packages and examples linked together, so you can run the following commands to start the development:

```bash
# pnpm
pnpm install
```

The postinstall script will build all the packages. In case you have errors, please manually build every package inside the folder `packages`.

# Known Issues

- Peers won't be able to connect with each other if either one of them is behind a VPN.
