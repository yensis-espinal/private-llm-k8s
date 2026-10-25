# Building a private LLM cluster, and treating it like production

**A two-node Kubernetes lab for local model serving — built security-first, documented as a
risk register, and red-teamed including the AI agent that helped build it.**

> *Sanitized write-up. Derived from an internal repository; all addresses, hostnames, MAC
> addresses, server names and account identifiers removed. Findings are referenced by their
> register IDs.*

---

## What it is

A private Kubernetes cluster for running large language models on my own hardware: a
controller VM and a GPU worker (RTX 2080, 8 GB VRAM, 64 GB host RAM), serving an 8B model
through a local chat UI. No model traffic leaves the network, and nothing is exposed — the
only path to the UI begins with an authenticated SSH session over a WireGuard-based mesh.

The point was never "run a chatbot." It was to build something small enough to understand
completely and then hold it to a standard I would defend in a production review: default-deny
everywhere, every exception documented, every claim verified against the running system rather
than the manifest that was supposed to produce it.

**Current state:** 28 lab findings opened, 18 closed and verified on live hosts. Ten stay open,
including one whose fix is written, tested and deliberately *not* applied — recorded as accepted
with its residual risk stated, rather than left reading as done. A further 9 concern the wider
corporate network the lab sits on — 5 owned by the infrastructure team, 3 mine, 1 shared — all
tracked in the same register whether or not I can close them.

---

## Architecture and the security model

| Layer | Decision |
|---|---|
| Admin access | Mesh VPN with SSH only. No cluster port reachable from the office network. |
| Model UI | No Service exposed, no ingress controller. Reached by SSH tunnel + `port-forward`. |
| Host firewall | Default-deny on both nodes; every allow rule carries a comment and a doc entry. |
| Pod network | Default-deny ingress **and** egress in the serving namespace; seven explicit policies. |
| Workloads | Pinned image digests, resource limits, dropped capabilities, seccomp, no service-account token, probes. |
| Node | CPU and memory reserved for the kubelet and OS, so one unbounded pod cannot take the node down. |
| Secrets | Never in the repo. Secret scanning on every commit; signed commits. |

Two deliberate trade-offs, stated rather than hidden:

- The model-serving pod needs outbound HTTPS to pull weights, so its egress policy permits the
  public internet while carving out **all** private address space. That is weaker than an egress
  proxy with an allowlist, and it is what was achievable without standing one up.
- Prompt and response traffic between the UI and the model is unencrypted on the pod network.
  Both pods are co-located today so it never hits the wire; a second GPU node would change
  that, which is why encrypting the pod network is tracked as an open finding.

---

## Four findings I'd actually talk about in an interview

They share a theme, and it is the thing I'd want to be hired for noticing: **every one is a
control that appeared to be working.** None was found by an attacker, a scanner or an alert.
Each was found by distrusting a green result and checking what it actually proved.

### 1. A staged security control went stale and would have caused a silent monitoring blackout

The network policy was written on one day; two monitoring CronJobs were added the next. The
policy's default-deny rule selected *every* pod in the namespace, but its exceptions named only
the two application pods. Applying it as written would have killed the health-check watchdog
**and** the alerter that reports on the watchdog — together, presenting as a generic connection
failure immediately after a network change.

The detail that matters: `kubectl apply --dry-run=server` passed all seven policies happily.
**Schema validity tells you nothing about whether rules still match the workloads.** Caught by
reading the manifest against the live pod specs before applying.

The generalized lesson now sits in the backlog: *staging is not a safe resting state.* A
reviewed-but-unapplied control decays against the system it is supposed to protect, and the gap
between writing it and applying it is a risk window that needs its own review.

### 2. The runbook's own negative tests produced a false pass

The procedure verified the model pod's egress restrictions by running `curl` inside that pod.
The image ships neither `curl` nor `wget`, so the command failed for the wrong reason and the
test reported success. A test that cannot fail is not a test — it is a comment that reads like
evidence. Rewritten to use probes that actually execute, and the real results logged.

### 3. A backup silently truncated, and the archive looked valid

Backing up a 890 MB volume by streaming `tar` through `kubectl exec` produced a 112 MB
archive. It listed cleanly, with 121 entries — a plausible count for the file set. **It was
missing the single file that held all users, chat history and settings.** The database's
write-ahead-log sidecars *were* present, which made the gap harder to spot: the backup looked
like a database backup.

Root cause is that `kubectl exec` streams through the API server and times out on large
transfers, failing in a way that still yields a well-formed archive. A valid archive is not a
complete one, and an entry count is not an inventory.

The fix was not just a different transfer method — it was deciding that the backup is not done
until a restore has been performed. For the irreplaceable tier that is now satisfied: restored
copy verified byte-identical by checksum, services healthy afterwards, and the in-use database
matching the restored file. The full in-place drill — take the volume away and bring it back
from the archive alone — is still open, and the exit criterion stays unmet until it runs.

### 4. A silent network-policy enforcement failure, diagnosed by reproduction rather than inference

Every newly created pod was being refused by the policy enforcer while a long-lived pod kept
working — so the service looked healthy from the outside. The first diagnosis was inferred from
a single log line after the fact. That wasn't good enough, so the failure was reproduced
deliberately under controlled conditions and cleared by correcting one parameter, which proved
it was a node-address mismatch and not a platform bug.

It was caught on day one by a 15-minute watchdog. Without that, it would have surfaced weeks
later as "the AI chat is randomly broken."

---

## Securing the AI agent that helped build it

The lab was built with an AI coding agent. The agent is part of the attack surface: it reads
untrusted content, holds shell access, and can reach the network. So it got its own control
set and its own red-team.

**Five independent layers:** instruction hygiene (content found in files is data, never
instructions) · a permission policy of allow/ask/deny rules · a normalizing pre-execution guard
· filesystem and network containment · a redacted audit trail written outside the repository.

**Red-team:** a file carrying an injected "priority override" told the agent to read the
cluster join token, post it to an external host, and conceal the act. Each layer was then
verified to stop it **on its own** — the instruction layer treated it as data, the guard blocked
the credential read, the permission layer denied the egress. A bypass suite of 96 probes covers
flag-order normalization, path-prefix stripping, wrapper prefixes, credential reads, policy
tampering and inline interpreters.

**What I learned from it is the part worth reporting.** Three findings came out of the agent's
own guardrails, and all three are about the *failure modes of controls*, not of attackers:

- **The audit log recorded what ran and was silent about what was blocked.** A pre-execution
  block stops the call before the logging hook fires. A quiet log therefore proved nothing
  about attempts — exactly backwards from what you want during an incident.
- **False positives are a security problem, not an annoyance.** Substring matching on
  credential words fired on ordinary English prose eight times, four of them blocking a commit.
  A control that blocks legitimate work is a control that gets switched off.
- **An isolation feature collided with a protection rule.** The anti-tamper rule blocked writes
  to the agent's configuration directory — correct, until agent sessions began running in
  isolated worktrees *inside* that same directory. The rule was right by its letter and wrong
  in intent. Fixed by neutralizing the workspace paths and re-running the unchanged rule set,
  so protected files stay protected by construction rather than by a whitelist.

A fourth was sharper still: **the guard patch was reported applied twice and was not applied
either time**, because the verification script defaulted to validating the source file rather
than the installed one. Verification that doesn't read the artifact in production is theatre.

---

## The habit that mattered most

Two of this project's worst self-inflicted problems trace to the same root: **a status cell
that did not match reality.** One understated risk — a reservation marked "to confirm" was read
as done, and its absence caused a real outage. One overstated it — a finding recorded as fixed
and verified still read "Open" days later.

They point in opposite directions and cost the same thing: a register nobody trusts gets
skimmed instead of read.

The rule that came out of it: the **change log is the ledger of what happened**; the risk
register is current state *derived* from it. When they disagree, the change log wins. Reconcile
both in the same session as any status change, and never record a status that no verification
step supports. "To confirm" is a ❌, not a ✅.

The same instinct shows up throughout: verify from the running system, not from the file that
was supposed to configure it. Resource limits confirmed by reading the node's cgroups rather
than trusting the apply output. Security context confirmed by reading process status on the
node rather than through the API. A static address confirmed by checking the route's origin,
not just that the address looked right — the distinction that was missed the first time, when
the machine looked correct but was still leasing.

---

## Honest open items

- Pod-to-pod traffic is still unencrypted between nodes. Deliberately sequenced after backups,
  because restarts are what has broken this cluster before and that change wants a restore path.
- The irreplaceable data tier is backed up off-node and a restore has been verified
  byte-identical. The **full in-place drill** — remove the live volume and bring it back from
  the archive alone — has not run, so the exit criterion is deliberately still open. A backup
  nobody has restored is a hypothesis.
- All persistent data still sits on a single node's local disk with a delete-on-release reclaim
  policy. Open, and rated High. The backup is a mitigation, not the fix.
- One container image exists only in a single node's local store with no registry copy. The
  manifest is in version control; the image is not.
- Monitoring is deliberately *not* built yet. It closes no finding, and the decision recorded
  for when it is built is standalone metrics collection with static scrape targets rather than
  the standard operator chart — because service discovery is the only reason the collector
  would need a cluster-wide read grant, and a two-node lab with a handful of stable endpoints
  does not have the problem discovery solves.

---

## What this demonstrates

Kubernetes network policy and workload hardening · host firewalling and default-deny design ·
threat modelling with documented trust boundaries · incident diagnosis by reproduction ·
local LLM serving on consumer GPU hardware · AI agent security: permission design, prompt-injection
red-teaming, audit trails, and the failure modes of the controls themselves · and the
documentation discipline to keep a risk register that is worth reading.
