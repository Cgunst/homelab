# Operations

## Monitoring

- A **dashboard** is for humans; it reads the monitoring system
  and hypervisor APIs. Where an integration couldn't authenticate with least-privilege
  credentials, the tile was removed rather than the credentials widened.

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

## Backups (honest status)

The monitoring VM has weekly consistent local recovery copies (app volume tarred with
the container stopped, config included, a handful retained, archives readability-tested).
That is **local recovery, not off-site DR**, and the lab-wide backup/restore-test
procedure is still draft — one of the known gaps listed here rather than glossed over.
