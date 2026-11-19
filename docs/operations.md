# Operations

How day-2 work actually runs here: what watches the lab, how patches go in, and how a
reboot window is executed. Described generically — no hostnames, no addresses, no
credential mechanics.

## Monitoring

- **Checkmk** (container on a dedicated VM) is the source of truth for state:
  hypervisor coverage via the platform's dedicated agent plus scoped read-only API
  tokens, Linux agents on guests, SNMPv3 for the switch, and active HTTP checks on
  service endpoints (health URLs, UI ports).
- A **dashboard** (also containerized) is for humans; it reads the monitoring system
  and hypervisor APIs. Where an integration couldn't authenticate with least-privilege
  credentials, the tile was removed rather than the credentials widened.
- A **health collector** script runs on a schedule: probes every host and service,
  diffs against the last run, writes timestamped reports, and exits non-zero on
  WARN/FAIL so the scheduler's job state itself is a signal.
- Certificate monitoring is layered: per-service expiry thresholds (with much tighter
  windows for a short-lived internal ACME leaf), plus a dedicated check that alerts if
  the CA's served leaf hasn't *rotated* recently — an expired cert is obvious, a
  stuck renewal is quiet.
- Known-accepted states (e.g. "no backup job on this VM") are **acknowledged**, never
  suppressed — the CRIT stays visible as a fact until it's actually fixed.

## Alerting

- Alert jobs deliver to a chat channel through the automation gateway; every retarget
  gets a test **through the same delivery path** the schedule uses, and the first
  scheduled run afterwards is confirmed, not assumed (see lesson "Alerting that
  silently wasn't").
- Delivery is only half the loop: job exit codes and delivery state get reviewed,
  because a silently failing alert job is indistinguishable from a quiet lab.

## Patch process (guests)

One guest at a time, in dependency-aware order (DNS late, and the machine you're
running from absolutely last):

1. **Record the "before" state** — versions of the packages in scope, and
   `systemctl is-active` for each service that must still be up afterwards.
2. **Snapshot first** — a named pre-patch snapshot on the hypervisor. The agent that
   does the work never deletes or rolls back snapshots; that's a human decision with
   a maintenance window.
3. **Upgrade conservatively** — `apt-get update && apt-get upgrade` (never
   `full-upgrade`/`dist-upgrade` in an automated pass), non-interactive frontend,
   confdef/confold so config prompts can't wedge the run. Held-back packages are
   *noted, not forced*.
4. **Verify** — fixed package versions where a CVE drove the run, every service from
   step 1 still active, plus functional probes (DNS resolves, endpoints return 200).
5. **Leave service restarts to the reboot window** where possible (needrestart reports
   what's stale; nothing restarts out from under a running workload mid-run).
6. **Log everything** per guest — one row per guest: upgraded, held back, before→after,
   snapshot name.

Node-level upgrades and reboots are a human-only action under a separate runbook.

## Reboot runbook (humans execute; agents prepare)

Prepared by the agent, executed by me:

- **Pre-checks:** guest patching done, boot flags set (`onboot` for guests that must
  return), no automation mid-flight, post-reboot check script tested *before* the
  window with its expected FAILs recorded (kernel version, cold-start-only memory
  change, services that only start at boot).
- **Order matters:** reboot the node whose guests are tolerated-down-first, then the
  other. Here that's the services/monitoring node first (it takes the agent, DNS,
  automation and NVR with it for a few minutes), then the inference node.
- **Expect slow boots where it's honest:** the large LLM service loads tens of GB
  before it answers health checks; the post-reboot check retries for up to 20 minutes
  instead of screaming on first contact.
- **Post-reboot check:** one script, PASS/FAIL per item, non-zero exit on any FAIL:
  kernel versions, every `onboot` guest running, per-guest services active, health
  endpoints, DNS resolution, monitoring and dashboards answering, memory sizes that
  only apply on cold start.
- **Cleanup afterwards, not during:** old kernels removed once the new one is proven
  (keep running + one fallback), pre-patch snapshots deleted only after the check
  passes — rollback (`pct rollback <guest> <snapshot>`) stays possible until then.

## Vulnerability management

Scan → enrich → rank → **verify applicability** → patch → re-scan. The scanner and the
correction stories (KEV misranking, host-kernel exposure, mixed-release inflation) are
in the [vuln-management-lab repo](https://github.com/Cgunst/vuln-management-lab).

## Backups (honest status)

The monitoring VM has weekly consistent local recovery copies (app volume tarred with
the container stopped, config included, a handful retained, archives readability-tested).
That is **local recovery, not off-site DR**, and the lab-wide backup/restore-test
procedure is still draft — one of the known gaps listed here rather than glossed over.
