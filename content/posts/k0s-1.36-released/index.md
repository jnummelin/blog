---
title: "k0s 1.36 Released: containerd 2.3, Safer etcd Scaling, and Expanded Platform Support"
date: 2026-06-12
draft: true
author: Jussi Nummelin (@jnummelin)
tags: ["k0s", "release", "kubernetes", "containerd", "etcd"]
cover:
  image: k8s-haru-480.png
  alt: "Kubernetes v1.36 Haru logo"
  caption: "Kubernetes v1.36 Haru release logo - Image from [Kubernetes v1.36 Release Announcement](https://kubernetes.io/blog/2026/04/22/kubernetes-v1-36-release/)"
---

We're excited to announce the release of k0s 1.36, now powered by Kubernetes 1.36. This release makes a meaningful leap in three areas: the runtime layer moves to containerd 2.3 with improved configuration handling and clearer upgrade guidance; the control plane becomes more resilient with a series of etcd reliability improvements that make multi-controller deployments safer to scale and recover; and platform flexibility grows with Traefik as an alternative node-local load balancing backend - bringing NLLB to Windows and ARMv7 nodes where Envoy was not an option.

k0s 1.36 continues our commitment to zero friction, zero dependencies, and zero cost. Let's dive into what's new!

---

## What's New in Kubernetes 1.36

Kubernetes 1.36 promotes several long-awaited features to stable and beta, and introduces new scheduling primitives for workload-aware coordination:

- **User Namespaces (GA)**: The `UserNamespacesSupport` feature gate graduates to generally available, giving pods strong host namespace isolation without requiring privileged containers.
- **Image Volumes (Stable)**: Mounting OCI images directly as volumes is now stable, enabling new patterns for delivering configuration, models, and other content as container images.
- **In-Place Pod Vertical Scaling at Pod Level (Beta, on by default)**: `InPlacePodLevelResourcesVerticalScaling` allows resizing CPU and memory for running pods at the pod level without restarts — now on by default.
- **Mutating Admission Policies (GA)**: `MutatingAdmissionPolicy` graduates to v1 and is enabled by default, giving cluster operators a CEL-based alternative to webhook-based mutation.
- **Gang Scheduling / Workload API (Alpha)**: A new `scheduling.k8s.io/v1alpha2` Workload and PodGroup API lands in the scheduler, enabling all-or-nothing scheduling for batch and ML workloads.
- **DRA Maturity**: Dynamic Resource Allocation sees multiple graduations — device taints and tolerations to beta, `DRAPrioritizedList` to GA, and `DRAAdminAccess` to GA — reflecting a maturing ecosystem for hardware-aware scheduling.
- **Strict IP/CIDR Validation (on by default)**: `StrictIPCIDRValidation` is now enabled by default; API fields no longer accept IPs with leading zeros or CIDRs with ambiguous host bits.
- **WebSocket Streaming to Kubelet (Beta, on by default)**: The API server now proxies `exec`, `attach`, and `portforward` WebSocket requests directly to the kubelet, improving streaming reliability.

For the full list of upstream changes, see the [official Kubernetes 1.36 release notes](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.36.md).

## What's New in k0s 1.36

Frame the k0s-specific changes around runtime upgrades, networking, reliability, and packaging foundations.

### Upgrade Notes

k0s 1.36 ships with **containerd v2.x**. This is the most important upgrade consideration in this release, and it requires planning if you use custom containerd configuration.

**Important:** containerd v2-format drop-ins are no longer supported in k0s 1.36. If your nodes still rely on old snippets in `/etc/k0s/containerd.d/`, upgrade those files to v3 format before production rollout.

Before upgrading, we strongly recommend this sequence:

1. Download the k0s 1.36 binary/package to your nodes.
2. With the new binary, run `k0s sysinfo` before restarting the k0s service, and fix any reported containerd drop-in issues.
3. Proceed with the k0s upgrade rollout (service restart) once `k0s sysinfo` is clean.

**containerd drop-in snippets must be v3 format** ([#7376](https://github.com/k0sproject/k0s/pull/7376)): If you have custom containerd configuration snippets in `/etc/k0s/containerd.d/`, they must use the containerd v3 config format. v2-format snippets will be rejected on startup. If you are unsure, run `k0s sysinfo` with the new 1.36 binary before restarting services — the new drop-in validation in sysprobes ([#7693](https://github.com/k0sproject/k0s/pull/7693)) will surface incompatible files early, before k0s attempts to start containerd.

**Config merging behaviour has changed** ([#7713](https://github.com/k0sproject/k0s/pull/7713)): k0s no longer applies a custom merge patch for containerd drop-ins. containerd 2.x natively deep-merges plugin sections from imported config files, which is the correct and supported approach. In practice this means drop-in files are now merged by containerd itself — the semantics are very similar but if you relied on the previous merge ordering, verify your resulting containerd config after the upgrade with `containerd config dump`.

### Runtime and Upgrade Experience

**containerd v2.3.1 is now the default runtime** ([#7566](https://github.com/k0sproject/k0s/pull/7566)). This brings a more robust container runtime with better support for modern OCI specs, improved performance, and a cleaner configuration model. The upgrade is straightforward for default setups, but clusters with customized containerd drop-ins should treat this as a planned migration step, not a blind in-place bump.

As part of this change, k0s now uses **containerd's native config merging** instead of its own merge pipeline ([#7713](https://github.com/k0sproject/k0s/pull/7713)). k0s defaults (sandbox image, Windows CNI paths) are written inline in the main config, and user drop-ins in `/etc/k0s/containerd.d/` are imported and merged by containerd itself. A filesystem watcher still triggers a containerd `SIGHUP` whenever a drop-in changes, preserving dynamic reconfiguration without any extra steps.

**Early drop-in validation** has been added to sysprobes ([#7693](https://github.com/k0sproject/k0s/pull/7693)). On node startup k0s now validates all containerd config snippets before attempting to launch containerd. This catches format errors — including the v2-to-v3 migration issues common when upgrading — immediately rather than leaving the node in a broken state.

**Platform-aware `airgap list-images`** ([#7459](https://github.com/k0sproject/k0s/pull/7459)): the `k0s airgap list-images` command now selects images for the requested target platform (`--platform`) instead of assuming the host. This means you can run the command on a Linux machine to generate the correct airgap bundle for a Windows or ARMv7 node, making offline deployments across mixed architectures much easier to prepare.

### Networking and Load Balancing

**Traefik as an alternative NLLB backend** ([#7405](https://github.com/k0sproject/k0s/pull/7405)): Node-local load balancing (NLLB) previously relied exclusively on Envoy, which is unavailable on ARMv7, RISC-V, and Windows. k0s 1.36 adds Traefik as a fully supported alternative. To use it, set `type: Traefik` in the `nodeLocalLoadBalancing` section of your cluster config:

```yaml
spec:
  network:
    nodeLocalLoadBalancing:
      enabled: true
      type: Traefik
```

Envoy remains the default for Linux amd64/arm64 clusters, but Traefik is now the recommended choice for Windows worker nodes and other platforms where Envoy is not available. This effectively brings NLLB to feature parity across all k0s-supported platforms.

**IPv6 single-stack promoted to beta and enabled by default** ([#7569](https://github.com/k0sproject/k0s/pull/7569)): The `IPv6SingleStack` feature gate moves from alpha to beta and is now on by default. Clusters running purely on IPv6 no longer need to pass `--feature-gates=IPv6SingleStack=true` on every controller. If you need to explicitly disable it, pass `--feature-gates=IPv6SingleStack=false`.

**kube-proxy rolls automatically on config changes** ([#7450](https://github.com/k0sproject/k0s/pull/7450)): Previously, changes to the kube-proxy ConfigMap would not trigger a DaemonSet rollout, leaving existing pods running with stale configuration until an unrelated restart happened. k0s now hashes the rendered kube-proxy config and embeds it in the pod template labels, so any configuration change triggers an automatic rolling update.

### Reliability and Resilience

k0s 1.36 addresses three etcd failure scenarios that have tripped up multi-controller deployments: joining a node with a bad peer URL, losing members after a backup restore, and dropping below quorum during a planned scale-down.

**New etcd members join as learners and are auto-promoted** ([#7629](https://github.com/k0sproject/k0s/pull/7629)): When a new controller joins the cluster, it is now added to etcd as a non-voting learner rather than immediately as a full voting member. The `EtcdMemberReconciler` monitors the learner and promotes it to a full member only once etcd reports it has caught up. This prevents a misconfigured joiner (for example, one advertising the wrong network interface) from immediately breaking quorum on the existing cluster — previously a single bad join on a two-node cluster was enough to stall the whole control plane.

**Stale `EtcdMember` objects are detected and reconciled** ([#7644](https://github.com/k0sproject/k0s/pull/7644)): If a controller was removed without going through the normal leave procedure — for example after a backup restore — its `EtcdMember` CR would remain marked as joined even though it was no longer present in the etcd cluster. k0s now cross-references CRs against the live etcd member list by member ID (more reliable than by name) and marks removed members as `Joined=false` automatically, without requiring `Spec.Leave` to be set.

**Controllers remove themselves from etcd before stopping** ([#7649](https://github.com/k0sproject/k0s/pull/7649)): Previously, when a controller was asked to leave the cluster, the `EtcdMemberReconciler` would wait until the node had already stopped before removing it from the etcd peer set. This could drop the cluster below quorum during the shutdown window — a particularly sharp edge when scaling down from two controllers to one. The leaving controller now removes itself from etcd while it is still running and can contribute to quorum, then hands off to the supervisor.

**Faster, cleaner API server shutdown** ([#7188](https://github.com/k0sproject/k0s/pull/7188)): k0s now enables the API server's watch termination grace period by default. Active watch streams are drained during shutdown rather than waiting for the full HTTP request timeout, which means the API server responds faster to init system stop requests under realistic cluster load. The supervisor stop timeout is derived automatically from the API server flags and clamped between 5 and 20 seconds to stay within typical systemd budgets.

### Packaging and Cross-Platform Foundations

**Embedded binaries are now packaged as an appended ZIP payload** ([#6770](https://github.com/k0sproject/k0s/pull/6770)): k0s has historically embedded its runtime binaries (kubelet, containerd, runc, etc.) using a custom bindata pipeline with pre-computed offsets baked into the binary at build time. In 1.36 this is replaced by appending a standard ZIP archive to the k0s executable at build time. The runtime reads directly from the appended ZIP and falls back to disk or `PATH` if no payload is attached.

This is largely a build infrastructure change with no user-visible behaviour difference in normal operation. The reason it's worth calling out is what it enables going forward: the ZIP model is a cleaner, more composable foundation for future packaging and distribution work — including post-build customization of the embedded asset set without recompiling the binary itself. It also makes Windows cross-compilation more straightforward: `make k0s.exe` and `make k0s.bare.exe` now work directly without setting `TARGET_OS=windows`.

## Component Updates

k0s 1.36 includes updates to all major runtime and networking components:

| Component      | Version |
|----------------|---------|
| Kubernetes     | 1.36.1  |
| etcd           | 3.6.12  |
| containerd     | 2.3.1   |
| runc           | 1.4.2   |
| Calico         | 3.32.0  |
| kube-router    | 2.10.0  |
| CoreDNS        | 1.14.3  |
| Konnectivity   | 0.36.0  |
| Metrics Server | 0.8.1   |
| Traefik (NLLB) | 3.7.5   |
| Envoy (NLLB)   | 1.37.2  |
| Helm SDK       | 3.21.1  |

## Release Statistics

The k0s 1.36 release represents a substantial effort from both the core maintainers and the broader community:

- **386 Pull Requests** merged to `main`
- **845 Commits** on `main`
- **19 Contributors** (non-bot PR authors)
- **9 New Contributors** joined the project:
  - [@olofvndrhr](https://github.com/olofvndrhr) in [#7151](https://github.com/k0sproject/k0s/pull/7151)
  - [@morlay](https://github.com/morlay) in [#7011](https://github.com/k0sproject/k0s/pull/7011)
  - [@smiggiddy](https://github.com/smiggiddy) in [#7247](https://github.com/k0sproject/k0s/pull/7247)
  - [@alek-thunder](https://github.com/alek-thunder) in [#7322](https://github.com/k0sproject/k0s/pull/7322)
  - [@martimmoura](https://github.com/martimmoura) in [#7329](https://github.com/k0sproject/k0s/pull/7329)
  - [@lethedata](https://github.com/lethedata) in [#7292](https://github.com/k0sproject/k0s/pull/7292)
  - [@dodgex](https://github.com/dodgex) in [#7384](https://github.com/k0sproject/k0s/pull/7384)
  - [@alliasgher](https://github.com/alliasgher) in [#7540](https://github.com/k0sproject/k0s/pull/7540)
  - [@luhenry](https://github.com/luhenry) in [#7606](https://github.com/k0sproject/k0s/pull/7606)

Thank you to everyone who contributed code, reviews, testing, and feedback!

## Get Started

Ready to try k0s 1.36? Here are the fastest paths to evaluate and adopt it.

**Download k0s 1.36:**

- [GitHub Releases](https://github.com/k0sproject/k0s/releases)
- Install script: `curl -sSLf https://get.k0s.sh | sudo sh`

**Documentation:**

- [k0s Documentation](https://docs.k0sproject.io/)
- [Quick Start Guide](https://docs.k0sproject.io/stable/install/)

**Join the Community:**

- [GitHub Discussions](https://github.com/k0sproject/k0s/discussions)
- [Slack Channel](https://kubernetes.slack.com/archives/C07VAPJUECS)
- [Community Office Hours](https://docs.k0sproject.io/stable/#community-hours)

## Closing

k0s 1.36 focuses on safer day-2 operations and broader platform coverage: containerd 2.3 with clearer upgrade behavior, stronger etcd lifecycle handling, and node-local load balancing options that extend support to more real-world environments.

If you're planning an upgrade, start by validating your containerd drop-ins and rollout process in staging, then move forward with confidence.

Thanks again to everyone who contributed to this release.
