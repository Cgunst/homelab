# Decision records

Short records of real choices in this lab, written as context → decision → why → trade-off.
Sanitized: role names instead of hostnames, documentation address space.

## 1. Internal ACME CA with a name-constrained intermediate

**Context.** Every internal service (firewall UI, hypervisor APIs, dashboards) was
serving self-signed certs. Browsers and API clients needed pinning or exemptions, and
certificate drift went unnoticed.

**Decision.** Run step-ca as an ACME server on a dedicated unprivileged LXC guest
(`ca.internal`). Keep the root CA offline-grade and issue a **constrained intermediate**:
the intermediate may only sign names under the `internal` DNS suffix and addresses in
the lab's documentation-style `10.10.0.0/16`-equivalent range, `pathlen: 0`. Wildcard
issuance is disallowed by policy. Services enroll via ACME with their own
`*.internal` name.

**Why.** Name constraints mean a leaked leaf or even a compromised ACME account can
never mint a certificate for a real internet domain — the policy limits the blast
radius cryptographically, not just procedurally. Automated renewal removes the manual
renewal treadmill.

**Trade-off.** The CA is now a critical dependency: loss stops all renewals. My backup
and restore procedure for it is still draft and untested, which I track as a known gap.
One service (the firewall GUI) stayed on a manually issued long-lived certificate after
I assessed the ACME plugin's challenge options — the plugin could alter firewall rules
during HTTP-01, which I judged worse than a manual renewal I can monitor for expiry.

## 2. Monitoring access to the hypervisor via an auditor-scoped API token

**Context.** Checkmk and the dashboard both need hypervisor data. The simplest setup —
root credentials — would hand a monitoring VM full control of both nodes.

**Decision.** Create a dedicated API user on each node with the built-in
**PVEAuditor role at `/`** (read-only), token-based, secrets stored 600 on-host and
mirrored only in protected stores.

**Why.** Monitoring is the service that gets breached first in a lot of attack paths.
Read-only at the API-role level means a compromised monitoring VM cannot start, stop,
or reconfigure anything.

**Trade-off.** Some newer integrations assume broader rights and just don't work: when
the dashboard's hypervisor widget couldn't authenticate against the platform's new
token-header format, I removed the widget instead of granting more rights, and got the
coverage through Checkmk's dedicated agent instead. Less convenient, smaller surface.

## 3. Big-model inference: GPU + system-RAM expert offload behind a systemd service

**Context.** The main LLM endpoint is a Mixture-of-Experts model whose weights are far
larger than the workstation GPU's VRAM — the service loads on the order of 88 GB into a
guest with 161 GiB of RAM, keeping the hot path on GPU and the MoE experts in system RAM.

**Decision.** Run it as `systemd` service (`Restart=on-failure`, `TimeoutStartSec=0`,
`LimitMEMLOCK=infinity`, env-file for the API key) that starts at boot, replacing the
previous pattern of an interactive tmux session. `LimitMEMLOCK=infinity` is required
for pinned memory in the GPU stack inside an LXC container.

**Why.** An interactive session dies with the terminal and never survives a node
reboot; the endpoint other things depend on (including the agent itself) needs a
unit with restart policy and journald logs.

**Trade-off.** Cold start is slow — several minutes of loading before the health
endpoint answers — so the post-reboot check retries health for up to 20 minutes instead
of assuming instant availability.

## 4. CPU-only model for camera descriptions, to keep the GPU free

**Context.** The NVR can ask an LLM to describe detected objects ("a person near the
driveway"). The obvious move was to reuse the GPU inference endpoint.

**Decision.** Point the NVR's GenAI integration at a **separate, CPU-only** guest
running a smaller model. The GPU stays dedicated to the main inference workload.

**Why.** Object descriptions are async, low-urgency text jobs — wrong tenant for the
only fast hardware in the house. CPU inference is good enough for a sentence or two
per event.

**Trade-off.** Slower responses, and I found the NVR's GenAI requests timing out
against the CPU endpoint (recent events stored `description=null`), so this path is a
work in progress — the timeout/appetite between NVR and CPU model needs tuning or a
faster small model.

## 5. LXC for service guests, VMs where the kernel is part of the job

**Context.** Proxmox offers both. Almost everything historically went to LXC.

**Decision.** Keep single-service Linux guests (DNS, CA, inference endpoints, NVR,
media) as LXC containers — cheap, snapshot-friendly, share the host kernel. Put anything
that needs its own kernel or emulates an appliance into a KVM VM: the monitoring stack
(Docker + its own kernel expectations) and Home Assistant OS are VMs. The NVR stays a
(privileged) LXC because it needs raw device passthrough (iGPU render node, Coral TPU)
which is simpler from a container.

**Why.** Containers are the right default for overhead; the exceptions are exactly the
places where kernel isolation or device access dominate the decision.

**Trade-off.** LXC guests share the host kernel, so kernel CVEs are **host exposure**
reported misleadingly against every guest — my vulnerability scanner has to model that
(see the `vuln-management-lab` case study). And privileged LXC is a real trust decision;
I document which guests are privileged and why.

## 6. No WAN port forwards; remote access only over the tailnet

**Context.** I want remote access to dashboards, the firewall UI, and services.

**Decision.** Zero port forwards on the WAN interface (verified read-only through the
firewall API: only the two automatic LAN anti-lockout rules exist). All remote access
rides a Tailscale tailnet.

**Why.** Nothing listening on the public internet is the best vulnerability
management. Every exposure decision gets made once, at the tailnet boundary, instead of
per-service.

**Trade-off.** No guest-passes-egress-from-anywhere — if the tailnet app is missing or
broken (phone swap, laptop dead), I'm locked out and need someone on-site. Accepted.

## 7. Monitoring VM: pinned cloud image + pinned container tags, not floating versions

**Context.** The monitoring stack was deployed from a Debian cloud image with Checkmk
and the dashboard in Docker.

**Decision.** Import a specific cloud image build after verifying its SHA-512 against
the published sums; pin container images to exact tags (and record the registry digest)
instead of tracking `latest`. An image-tag upgrade is an explicit, reversible change:
record the current digest first so rollback is `docker compose` against the old digest.

**Why.** Monitoring that dies silently because an unattended image update broke is
worthless; and with a recorded digest, "undo the upgrade" is mechanical instead of
creative.

**Trade-off.** Upgrades need manual lift. That lift turned out to hide real problems —
one vendor image upgrade crash-looped because of my volume layout and got rolled back
(see `docs/lessons.md`). Pinned + rolled back beats floating + broken.

## 8. Two separate hypervisor installs, deliberately not clustered

**Context.** Two nodes, each with its own Proxmox install (distinct cluster CA
identities — discovered, not assumed).

**Decision.** Keep them unclustered: independent APIs, independent auth, guests
migrated only manually. Every operational doc must name the node with any VMID.

**Why.** A cluster joins failure domains (quorum, shared auth, correlated updates).
The two nodes don't need live migration; the workload fits each node.

**Trade-off.** VMIDs collide between nodes — CT 100 exists on **both** — which has
already caused near-miss automation mistakes, so tooling and docs carry the node name
everywhere (see `docs/lessons.md`).
