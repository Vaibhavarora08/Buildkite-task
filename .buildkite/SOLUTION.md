# Buildkite practical exercise — Vaibhav Arora

## Outcome
Three-step pipeline working end-to-end. Final job log prints 
`Hello, Vaibhav Arora!` as required. Screenshot attached in email.

Repo: [your forked repo URL]

---

## Approach
Read the source code before touching infrastructure. Forked the repo 
rather than using the UI editor to experience real webhook-triggered 
builds — that's how a customer would use it, and it surfaces real 
behaviour like queue configuration and build attribution.

Broke the work into: infrastructure first, pipeline second, 
iterate on errors third.

---

## Infrastructure choices

| Decision | Choice | Why |
|---|---|---|
| EC2 access | SSM Session Manager, no SSH | Closed port 22 entirely — no key pairs, full audit trail |
| OS | Amazon Linux 2023 | AWS-native, SSM pre-installed, better security defaults |
| Instance | t3.micro on-demand | Stable for demo — Spot risks termination mid-presentation |
| Subnet | Default public subnet | Right-sized for a single demo agent |
| EC2 state | Stopped when not in use | Cost control — ephemeral agents solve this properly in production |

---

## Three things worth calling out

**The `$$NAME` escaping**  
Buildkite interpolates `$VAR` at parse time, not shell runtime. 
Switching to `$$NAME` fixed an empty value bug. Easy to miss — 
worth flagging to customers migrating from other CI tools.

**`mount-buildkite-agent: true`**  
From the Docker plugin docs. Without it, `buildkite-agent` isn't 
available inside the container and artifact upload fails silently. 
Customers doing Docker-based builds will hit this immediately.

**Block step has no native timeout**  
`timeout_in_minutes` doesn't apply to block steps — a blocked build 
waits indefinitely. Production workaround: watchdog job calling the 
REST API to cancel after N minutes. Worth knowing for customer conversations.

---

## What I'd do differently in production

Ephemeral agents instead of a persistent EC2, artifact storage delegated 
to a customer-owned S3 bucket with KMS, SSM Parameter Store instead of 
Buildkite metadata for anything sensitive, private subnet with VPC 
endpoints so build traffic never hits the public internet.

---

## What I simplified and why

Skipped golden Docker image, binary signing, and private subnet — each 
adds real production value but would have been over-engineered for a 
demo. Noted where each would apply in a customer context.

---

## Note on pipeline.yml
Initial structure drafted with AI assistance. All infrastructure 
decisions, debugging, and fixes worked through independently.