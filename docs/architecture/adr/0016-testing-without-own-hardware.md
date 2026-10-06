# ADR-0016: Testing without own hardware — simulated provider, container nodes, real-VM tier

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0016-testing-without-own-hardware.ru.md)

## Context

The spec demands unit, integration, API, provider, engine, compatibility, component and E2E tests, a test environment (kind, k3d, Vagrant, Docker, Terraform test environments), a mandatory E2E scenario and failure testing (prompt §61–63). The owner has **no own machines** (answer to the Phase 0 questions). Without real hosts we still need strong confidence in SSH bootstrap, kubeadm and add-on installation.

## Decision

Use three test tiers plus local clusters:

1. **Simulated provider** (every PR): fake hosts with configurable facts, latencies and fault scripts. The real engine, API, SSE, UI and CLI run on top of it. It proves orchestration logic, failure handling, resume and crash recovery. It is marked `TestOnly` and refused in production profiles.
2. **Container nodes** (PR subset + nightly): privileged containers with systemd + sshd per supported OS. The platform uses **real SSH, the real bootstrap layer, real kubeadm, real Cilium and add-ons**, the same technique `kind` uses to run kubeadm in containers. Known limits (kernel modules and sysctls belong to the host; no real L2 or disks) are handled with explicit test-profile capabilities and documented.
3. **Real VMs** (nightly/manual) once a budget exists: a Hetzner Cloud project with a CI token and a spend cap (recommended), or nested QEMU/KVM VMs on CI runners when `/dev/kvm` is available, or the owner's or contributors' machines via `make e2e-real`.
4. **kind / k3d** clusters (`make test-cluster`) for add-on and Helm-service tests.

Phase reports must state honestly which tiers verified a feature. Nothing is called "tested on bare metal" without Tier-3 evidence.

## Alternatives considered

- **Vagrant + VirtualBox/libvirt locally**: heavy, slow, hard to run in CI, and not available in this cloud environment. Kept optional for contributors.
- **Only simulated tests**: insufficient confidence for OS/kubeadm behaviour.
- **Only real VMs**: needs money and time on every PR, and slow feedback.

## Consequences

- Positive: most real-world bootstrap bugs are caught without any cloud bill. Tier 1 is fast enough for every PR.
- Negative: container nodes don't reproduce kernel, network and disk behaviour faithfully, so some bugs only show on Tier 3. A small cloud budget is needed before the MVP gate (flagged as a project risk).

## References

- [Testing strategy](../testing-strategy.md)
