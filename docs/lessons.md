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
