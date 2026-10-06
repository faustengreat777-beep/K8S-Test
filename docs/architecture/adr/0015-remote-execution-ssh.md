# ADR-0015: Agentless remote execution over SSH with typed commands and an ephemeral signed node helper

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0015-remote-execution-ssh.ru.md)

## Context

Bare-metal and VPS provisioning needs remote execution on hosts we don't control: OS preparation, kernel modules, sysctl, container runtime, Kubernetes packages, kubeadm init/join (prompt §52–53). The spec requires private key, password, bastion/jump host, proxy, SSH agent and host verification support. It forbids storing private keys in plaintext and forbids inserting user input into shell commands (`exec.Command("sh", "-c", userInput)`, prompt §54, §116). Research for Phase 0 also showed real host heterogeneity: Ubuntu 26.04 ships Rust coreutils and `sudo-rs`, EL10 requires x86-64-v3, distro containerd packages are too old, and SELinux is enforcing on the EL family.

## Decision

- **Agentless**: no permanently installed agent on nodes. The platform connects over SSH only during operations (and for health probes that need host access).
- **SSH client**: `golang.org/x/crypto/ssh` with `agent`, `knownhosts` and SFTP (`pkg/sftp`). It supports keys, passwords, agent, **jump-host chains**, HTTP/SOCKS proxy, keep-alives, pooling and per-command timeouts.
- **Mandatory host-key verification**: known_hosts, SSH CA, or TOFU with explicit user confirmation of the fingerprint (stored per org). A mismatch stops the operation (`SSH_HOST_KEY_MISMATCH`).
- **Typed commands only**: `Command{Program, Args[], Env(allow-listed), Stdin, Sudo, Timeout}`. Each argument passes a type-specific validator and is quoted with strict POSIX single quotes. A custom static analyzer in CI forbids `sh -c` patterns and string-built programs.
- **Files over SFTP**: configs are rendered from Go structs with real encoders (YAML/TOML/INI) and uploaded atomically (temp + fsync + rename) with explicit mode and owner.
- **Ephemeral signed node helper (Phase 3)**: a small static Go binary (amd64/arm64), uploaded per operation, verified by checksum, and invoked as `helper <step>` with **JSON on stdin and JSON on stdout**. It does facts collection, file operations, package-manager calls (argv-based), service management and checks. It is removed afterwards. This avoids dependence on GNU-specific shell tooling and makes node steps unit-testable in Go. The helper never accepts arbitrary scripts.
- **Privilege**: `sudo -n` for specific steps. A dedicated provisioning user with restricted sudo is recommended. Root login is not required.
- **Secrets**: keys are decrypted just in time in worker memory and never written to disk. Agent forwarding is off by default.

## Alternatives considered

| Option | Why not |
|---|---|
| Ansible (playbooks over SSH) | Python runtime on the worker, weak typing and error classification, templated YAML/Jinja is an injection surface, harder to stream structured progress, a second execution model next to the engine |
| Permanent node agent (pull model) | Better for NAT'd or edge nodes, but needs agent lifecycle, upgrades, its own auth/PKI and a larger attack surface on every node. It may come later as an *optional* transport for edge/SaaS (outbound-only), behind the same `NodeHandle` interface |
| Raw shell scripts (curl \| bash style) | Exactly what the spec forbids: untyped, fragile, injection-prone |
| cloud-init only | Works only at instance creation and only on clouds, not for existing hosts. It will be used *in addition* by cloud providers for first-boot setup |

## Consequences

- Positive: a small attack surface on nodes, typed and testable node operations, explainable errors from structured helper output, a uniform model for bare metal and cloud VMs.
- Negative: the helper binary must be built, signed and distributed per architecture, which adds to the release process. SSH connectivity from workers to nodes is required. The engine handles NAT'd nodes only through jump hosts until an optional agent transport exists.

## References

- [Security model §7](../../security/SECURITY_MODEL.md), [Core interfaces §5](../core-interfaces.md), [Provisioning engine §10](../provisioning-engine.md)
