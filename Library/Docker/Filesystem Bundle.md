# Container Format
This section defines a format for encoding a container as a _filesystem bundle_ - a set of files organized in a certain way, and containing all the necessary data and metadata for any compliant runtime to perform all standard operations against it.

The definition of a bundle is only concerned with how a container, and its configuration data, are stored on a local filesystem so that it can be consumed by a compliant runtime.

A Standard Container bundle contains all the information needed to load and run a container. This includes the following artifacts:
1. `config.json`: contains configuration data. This REQUIRED file MUST reside in the root of the bundle directory and MUST be named `config.json`. See [`config.json`](https://github.com/opencontainers/runtime-spec/blob/20a2d9782986ec2a7e0812ebe1515f73736c6a0c/config.md) for more details.
2. container's root filesystem: the directory referenced by [`root.path`](https://github.com/opencontainers/runtime-spec/blob/20a2d9782986ec2a7e0812ebe1515f73736c6a0c/config.md#root), if that property is set in `config.json`.

When supplied, while these artifacts MUST all be present in a single directory on the local filesystem, that directory itself is not part of the bundle. In other words, a tar archive of a _bundle_ will have these artifacts at the root of the archive, not nested within a top-level directory.

