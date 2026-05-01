# Buildkite Pipeline — Practical Exercise

## Overview

A three-step Buildkite pipeline that builds a Go binary inside a Docker 
container, accepts user input via a block step, and executes the binary 
with that input.

**Output:** `Hello, [your name]!` printed in the job logs.

---

## Pipeline Steps

**Step 1 — Build**  
Compiles the Go application inside a `golang:1.18.0` Docker container 
using the Buildkite Docker plugin. The resulting binary (`hello-bin`) is 
uploaded to Buildkite's artifact store.

**Step 2 — Block**  
Pauses the pipeline and prompts the user to enter their name via a form 
field. The value is stored in Buildkite metadata (`user-name`).

**Step 3 — Run**  
Downloads the binary from the artifact store, retrieves the name from 
metadata, and executes `./hello-bin "[name]"`.

---

## Infrastructure

| Component | Choice | Reason |
|---|---|---|
| Cloud | AWS EC2 (t3.micro, on-demand) | Free tier, stable for demo — Spot risks mid-demo termination |
| OS | Amazon Linux 2023 | AWS-native, better security defaults, SSM pre-installed |
| Access | SSM Session Manager (no SSH) | No open port 22, no key pairs, full CloudTrail audit trail |
| Storage | gp3 EBS, 8GB | Current AWS default; artifacts don't persist on disk |
| Metadata | IMDSv2 only, hop limit 1 | SSRF protection; prevents container escape to metadata endpoint |
| Bootstrap | EC2 User Data | Repeatable — Docker + Buildkite agent install on launch, no manual steps |

Security group: **no inbound rules**. Outbound HTTPS only (SSM, Buildkite agent, Docker pulls).

---

## Assumptions

- A free-tier AWS account (`t3.micro`) is sufficient for a single agent running this workload
- The Buildkite cluster uses a named queue (`test-buildkite`) — the agent config must 
  declare `tags="queue=test-buildkite"` to match
- Artifacts are stored in Buildkite's managed store (acceptable for this exercise; 
  production would delegate to a customer-owned S3 bucket)
- Buildkite metadata is appropriate for non-sensitive user input (name only — 
  not secrets or PII)

---

## Key Decisions

**Docker plugin over raw `docker run`**  
Keeps the pipeline declarative. Any agent with Docker can run this build — 
no Go installation required on the host.

**`mount-buildkite-agent: true`**  
Required so `buildkite-agent artifact upload` works inside the container. 
The plugin mounts the host binary into the container at runtime.

**`$$NAME` not `$NAME`**  
Buildkite interpolates `$VAR` at parse time (before the shell runs). 
`$$` escapes this and defers resolution to the shell at runtime.

**Artifact path matching**  
Upload path (`/workspace/hello-bin`) and download path (`workspace/hello-bin`) 
must match exactly as Buildkite stores them.

---

## What I Would Do Differently in Production

| Demo | Production |
|---|---|
| Buildkite artifact store | Customer-owned S3 bucket with KMS encryption + binary signing |
| Buildkite metadata | AWS SSM Parameter Store (persistence, IAM control, audit log) |
| Single on-demand agent | Ephemeral agents — spin up per build, Spot-viable |
| Default VPC | Private subnet + NAT Gateway + VPC endpoints for S3 |
| Manual trigger | Webhook on `git push` |

---

## Troubleshooting Notes

**Queue mismatch** — Buildkite clustered mode requires an explicit queue tag 
in the agent config. Added `tags="queue=test-buildkite"` to 
`/etc/buildkite-agent/buildkite-agent.cfg`.

**`buildkite-agent` not found inside container** — Resolved with 
`mount-buildkite-agent: true` in the Docker plugin config.

**Binary name conflict** — Source lives in `hello/`; output named `hello-bin` 
to avoid a conflict between the directory and the binary.

**Variable interpolation** — Switched from `$NAME` to `$$NAME` to defer 
resolution to shell runtime.