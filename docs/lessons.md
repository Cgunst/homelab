# Lessons learned

Things that actually broke, misled, or nearly bit me — each one with what changed
afterwards. Sanitized to role names and documentation addressing.

## 1. VMIDs collide across two separate hypervisor installs

Two Proxmox nodes, two independent installs, no cluster — which means **CT 100 exists
on both nodes** (one service guest on each, totally unrelated). Every `pct` command is
node-local, so an automation that "just knows CT 100" will patch, snapshot, or shut
down the wrong guest on the wrong node.

**Fix:** every doc, script, inventory row, and runbook line carries the node name next
to the VMID (`CT 100 @ pve-main` vs `CT 100 @ pve-mini`), and remote execution always
addresses the node first (`ssh <node> pct exec <id> …`). Boring, but it was a real
near-miss class of bug, not a hypothetical.

## 2. Alerting that silently wasn't

Daily and weekly alert jobs were configured to deliver to a chat platform **that was
never actually configured** on the agent gateway. The jobs ran, produced reports, and
"delivered" — with delivery failing every time, visible only if you read job state
tables. Three recurring alerts went nowhere for days.

**Fix:** retargeted every job to a platform that exists and verified with a test
message through the *same delivery path the cron engine uses*, not a side channel.
Lesson: an alert pipeline is only proven end-to-end; the thing most likely to be
broken is the delivery of the alert about the thing being broken. Delivery failures
themselves are now something I check on every monitoring review.

## 3. Stale apt lists made "fully patched" a lie

An update report counted upgradable packages with `apt list`. One guest reported
**zero** upgradable — and was actually the most out-of-date machine in the lab. Its
package lists were months old, and several other hosts had 3–7 week old lists. Worse,
one scan produced a confidently wrong escalation from this: a "still vulnerable to an
OpenSSH CVE, therefore unpatched for ~2 years" claim that turned out to be a
**stale-database false positive** — the pre-patch capture showed the package was
already at or above the fixed version.

**Fix:** refresh lists before trusting any count (`apt update` first, or fail loudly);
record the list freshness (`ls -l /var/lib/apt/lists` mtime) next to every count; and
verify "vulnerable" claims against package versions directly, not against cached
metadata. Counts without a freshness stamp are now treated as unknown, not as good news.

## 4. A half-finished release upgrade inflated findings by an order of magnitude

One guest's `/etc/os-release` said the current Debian release, but its installed
packages and apt state were still from the previous release — a release upgrade someone
started and never finished. A scanner dutifully compared old-release package versions
against new-release advisories and produced **21,846 findings** for a guest whose real
issue was the mixed-release state itself, not 21k unique exploits.

**Fix:** the scanner now detects mixed-release dpkg states (majority of package
version strings disagree with os-release) and flags the guest *instead of* reporting a
count. The guest's actual remediation (rebuild vs finish the upgrade, with the
application's Python runtime pinned in mind) is an in-progress item with a written
runbook and recommendation.

## 5. Kernel CVEs inside containers are host exposure, not guest findings

All LXC guests share the hypervisor kernels, and each guest reports `linux-libc-dev`
(kernel headers). Scanners therefore attributed the same kernel CVEs to every container
as if each ran its own kernel. Half my early "top findings" were the same handful of
kernel issues multiplied by guest count.

**Fix:** the scanner moves `linux-libc-dev` findings into a separate **host-kernel
exposure** section, checked against the *running* kernel of each node. Remediation
changes meaning too: it's a node reboot into the already-installed kernel, not anything
inside the guests. A reboot window is planned; until then that section stays open by
design rather than being silently counted as fixed.

## 6. The config file on disk wasn't the config in use

The NVR's on-disk config showed one fake test camera; the running service reported two
real cameras and an active TPU detector. The file I'd have backed up, edited, or
restored was a dead sample shipped in the application's source tree.

**Fix:** read the running config from the service's own API, and trace where the
software actually loads from — here it was a constant hardcoded in the application
package (`CONFIG_DIR = "/config"`), with no environment override set. I recorded the
real config path, verified which paths survive restarts (persistent subvolume vs
tmpfs), and noted that only *restart* survival was proven — a rebuild-time restore
list comes from that trace, not from whatever config file is lying around.

## 7. A pinned image upgrade crash-looped because of my volume layout

A patch-level upgrade of the monitoring server (new container image tag, recorded
digest for rollback) crash-looped instantly: `omd: command not found`. The site's data
volume was mounted over **all of** `/omd` — which in the new image is a symlink into
`/opt/omd` — so the volume shadowed the image's freshly-installed version tree. My
volume layout worked by accident with the old image and broke on the new one.

**Fix:** rolled back to the recorded digest (healthy again within minutes, all monitored
hosts and TLS trust config intact — verified after rollback, not assumed). Root cause
recorded with two fix options; the volume-layout change is in progress because it needs
its own maintenance pass. Lesson: a volume mounted at a directory that a newer image
turns into a symlink is a time bomb — pin digests, and treat rollback as part of the
upgrade plan, not an emergency idea.
