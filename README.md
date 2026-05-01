# gVisor (gvisor)
gVisor is an application kernel written in Go that implements a substantial portion of the Linux system surface. It provides an additional layer of isolation between running applications and the host operating system, intercepting and handling application system calls in user space to reduce the attack surface of the host kernel.

**URL:** [Visit APIs.json URL](https://raw.githubusercontent.com/api-evangelist/gvisor/refs/heads/main/apis.yml)

## Scope

- **Type:** Index
- **Position:** Consuming
- **Access:** Open Source

## Tags:

 - Containers, Kernel, Linux, Open Source, Sandboxing, Security

## Timestamps

- **Created:** 2026-03-26
- **Modified:** 2026-04-28

## APIs

### gVisor
gVisor is an open-source application kernel written in Go that provides an additional layer of isolation between containerized applications and the host operating system. It implements a substantial portion of the Linux system call interface in user space, making it compatible with most Linux applications while providing stronger security guarantees than traditional container runtimes. gVisor does not expose REST or gRPC APIs; integration is via the OCI runtime interface (`runsc`) used by Docker and Kubernetes.

**Human URL:** [https://gvisor.dev/](https://gvisor.dev/)

#### Tags:

 - Containers, Kernel, Linux, Open Source, Sandboxing, Security

#### Properties

- [Documentation](https://gvisor.dev/docs/)
- [Getting Started](https://gvisor.dev/docs/user_guide/quick_start/docker/)

## Common Properties

- [Website](https://gvisor.dev/)
- [GitHub Organization](https://github.com/google)
- [GitHub Repository](https://github.com/google/gvisor)
- [Documentation](https://gvisor.dev/docs/)
- [Blog](https://gvisor.dev/blog/)

## Maintainers

**FN:** Kin Lane

**Email:** kin@apievangelist.com
