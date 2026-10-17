# ADR-0001: Use the Host Environment with User-Level Permissions

- **Status:** Approved

## Context

The project requirements specify that the work must be carried out inside a virtual machine. However, the available machine has limited resources, so three possible approaches were considered:

1. Create and use a virtual machine as required by the project.
2. Install and run tools on the host within the permissions available to the current user, without relying on `root` or `sudo` privileges.
3. Use a cloud environment to access administrative-level privileges for system management.

## Decision

Run the project on the host using the permissions available to the current user. Where supported, install external tools as standalone single-binary executables to reduce reliance on system-level installation and administrative privileges.

## Justification

- Reduce the latency introduced by running the environment inside a virtual machine.
- Use only the machine resources that are necessary.
- Start experimenting and developing immediately without setting up an additional cloud environment.

## Consequences

### Pros

- Development and experimentation can begin immediately.
- Reduces the latency introduced by running the environment inside a virtual machine.
- Makes the environment and initial setup easier to understand.

### Cons

- Under the current permission constraints, K3d may not be usable because it depends on Docker or a container runtime that requires sufficient permissions.
- Dependencies required to start the project must be found and prepared to work within user-level permissions. Standalone single-binary tools should be preferred when available.
- Using the host instead of a virtual machine may conflict with the project requirements and could affect project evaluation.
