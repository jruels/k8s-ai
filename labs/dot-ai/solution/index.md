# Lab: dot-ai — An Agentic DevOps Platform

## What you will learn

- What `dot-ai` (the DevOps AI Toolkit) is, and how an "agentic platform" differs
  from a copilot or an in-cluster agent
- How to deploy the dot-ai server (with its Qdrant vector database) via Helm
- How to drive it with the CLI: natural-language `query` and agentic `remediate`
- How **manual vs. automatic** remediation works, and why gated execution is the
  heart of safe agentic operations
- *Why* you would give a tool a goal instead of a command

This is one of three labs on AI tooling for Kubernetes, and the most ambitious.
The **kubectl-ai** lab put a copilot in your shell; the **kagent** lab put agents
in your cluster; this lab introduces a platform you hand *goals* to.

---

## Why dot-ai?

### From "answer my question" to "achieve my goal"

The first two tools answer questions: *"what pods are unhealthy?"* `dot-ai` (by
Viktor Farcic) is built around a bigger ambition — you give it a **goal in plain
English** (*"deploy a database"*, *"figure out what's broken and fix it"*) and it
discovers what your specific cluster can do, plans an approach, runs a multi-step
investigation, and **proposes or (if you allow it) executes** the changes.

### Two design choices that define it

**1. It is built around MCP (the Model Context Protocol).** dot-ai's primary,
intended use is as an MCP *server* that an AI coding assistant (Claude Code, Cursor,
and similar) drives in a conversation. The standalone **CLI you will use in this lab
is the headless equivalent** — the same capabilities without needing an assistant —
which makes it ideal for a lab and for automation/CI. Understanding this matters:
dot-ai is designed to be the "tools and brains" that a larger AI workflow plugs
into, not just a one-off command.

**2. It is cluster-*aware*, not just cluster-readable.** dot-ai deploys a **Qdrant
vector database** and indexes your cluster's *capabilities* (what kinds, operators,
and CRDs are available) into it. That means its recommendations are grounded in
what **your** cluster can actually do — not generic internet advice. This semantic
memory is why the platform is heavier than the other two tools: it is doing more
than reading live state, it is building and searching an understanding of your
environment.

### The mental model

If kubectl-ai is "autocomplete in my shell" and kagent is "a team of agents in my
cluster," dot-ai is **an AI platform engineer you delegate goals to** — one that
investigates, validates its plan (it even dry-runs proposed changes), and then
*asks permission* before touching anything.

---

## Prerequisites

You need a running single-node Kubernetes cluster and an OpenAI API key.

Start (or restart) minikube with extra memory — dot-ai deploys a server plus a
Qdrant vector database:

```bash
minikube delete
minikube start --memory=6144
```

Set your OpenAI key in this shell:

```bash
export OPENAI_API_KEY="<your-openai-api-key>"
```

### Create a workload to investigate

Deploy one healthy app and one deliberately broken one so dot-ai has a real problem
to diagnose and remediate:

```bash
kubectl create deployment nginx --image=nginx
kubectl create deployment broken --image=nginx:doesnotexist
```

Confirm the broken Pod is stuck in `ImagePullBackOff`:

```bash
kubectl get pods
```

```
NAME                      READY   STATUS             RESTARTS   AGE
broken-7b58495bf7-xxxxx   0/1     ImagePullBackOff   0          20s
nginx-56c45fd5ff-xxxxx    1/1     Running            0          20s
```

---

## Step 1 — Deploy the dot-ai server

dot-ai is model-agnostic; we point it at OpenAI. It authenticates CLI calls with a
**bearer token**, so we generate one and save it for later. Qdrant is bundled with
the Helm chart and deployed automatically.

```bash
# A random bearer token the server and CLI will share; save it for later
export DOT_AI_AUTH_TOKEN=$(openssl rand -base64 32)
echo "$DOT_AI_AUTH_TOKEN" > ~/dot-ai-token.txt

helm upgrade --install dot-ai \
  oci://ghcr.io/vfarcic/dot-ai/charts/dot-ai \
  --namespace dot-ai --create-namespace \
  --set ai.provider=openai \
  --set ai.model=gpt-4.1 \
  --set secrets.openai.apiKey="$OPENAI_API_KEY" \
  --set secrets.auth.token="$DOT_AI_AUTH_TOKEN" \
  --timeout 6m --wait
```

**Why `--wait` and a 6-minute timeout?** The server image is large and Qdrant
needs a persistent volume, so the first install can take a few minutes. `--wait`
tells Helm to block until every component is actually `Running` and the server's
health check passes — so when the command returns, you know it is ready.

**Why `ai.provider=openai`?** dot-ai defaults to Anthropic. These two `--set`
flags (`ai.provider` and `ai.model`) plus the matching `secrets.*.apiKey` are all
that change to use a different LLM backend.

**Production note on secrets.** Passing the key with `--set secrets.openai.apiKey=...`
is fine for this lab and matches the upstream docs, but the value then lands in your
shell history and in `helm get values dot-ai -n dot-ai`. In production, pre-create the
Kubernetes Secret yourself (or use sealed-secrets / an external-secrets operator) and
point the chart at it via `secrets.name`, rather than passing the raw key on the
command line.

Confirm the three components are up:

```bash
kubectl get pods -n dot-ai
```

You want `dot-ai`, `dot-ai-agentic-tools`, and `dot-ai-qdrant-0` all `Running`.

**What are these three Pods?** `dot-ai` is the MCP/REST server (the brain),
`dot-ai-agentic-tools` is the sandbox where it executes `kubectl`/`helm` tool
calls, and `dot-ai-qdrant-0` is the vector database holding its semantic memory of
your cluster.

---

## Step 2 — Install and connect the CLI

The CLI is a thin client that talks to the server you just deployed. Install it:

```bash
sudo curl -sL https://github.com/vfarcic/dot-ai-cli/releases/latest/download/dot-ai-linux-amd64 \
  -o /usr/local/bin/dot-ai && sudo chmod +x /usr/local/bin/dot-ai
```

Expose the server with a port-forward in **one terminal** (leave it running):

```bash
kubectl port-forward -n dot-ai svc/dot-ai 3456:3456
```

In a **second terminal**, tell the CLI where the server is and how to authenticate,
then confirm connectivity. Persist the server URL once in the CLI's own config file
(`~/.config/dot-ai/settings.json`), and keep the bearer token in an environment variable:

```bash
# Persist the server URL (written to ~/.config/dot-ai/settings.json)
dot-ai config set server-url http://localhost:3456

# The auth token is read from the environment, NOT stored in the config file
export DOT_AI_AUTH_TOKEN=$(cat ~/dot-ai-token.txt)

dot-ai version
```

You should see `status: success` and a list of platform capabilities such as
`ai-recommendations`, `semantic-search`, and `capability-scanning`.

**Why config for the URL but an env var for the token?** `dot-ai config set` persists
non-secret settings (server URL, output format) so you don't retype them every session —
run `dot-ai config list` to see them. The **token is a secret**, so dot-ai deliberately
does *not* offer a config key for it; you supply it via the `DOT_AI_AUTH_TOKEN`
environment variable (or a `--token` flag) instead, keeping the secret off disk. A flag
or env var always overrides the persisted config.

**Why a separate server and CLI?** This split is the whole point of dot-ai's
design: the *server* is a long-lived, shared brain (the same one an AI assistant
would connect to over MCP), and the *CLI* is just one of several possible clients.
In production you would expose the server through an Ingress and point many clients
at it; in this lab a port-forward is the simplest equivalent.

---

## Step 3 — Ask a question in plain English

`dot-ai query` takes a natural-language intent and runs an agentic investigation
against your cluster:

```bash
dot-ai query "How many pods are in the default namespace and are any of them unhealthy? If so, explain why."
```

In the `result` you will see `toolsUsed: [kubectl_get, kubectl_describe]` and a
`summary` that correctly identifies `broken-…` as `ImagePullBackOff` caused by the
missing `nginx:doesnotexist` image. Note the `iterations` count — the platform ran
several tool-calling rounds before it was confident enough to answer.

This is the same read-tools-in-a-loop pattern you saw with kubectl-ai and kagent.
What differs is everything *around* it: the loop runs on a shared server, grounded
by the cluster-capability index in Qdrant, and is exposed the same way to a CLI or
to an AI assistant over MCP.

---

## Step 4 — Let it plan a fix: agentic remediation

This is where dot-ai goes beyond the other tools. Ask it to remediate the issue in
**manual mode** — it will investigate and *propose* a fix but **will not execute
anything without your approval**:

```bash
dot-ai remediate \
  --issue "The broken deployment in the default namespace has a pod stuck in ImagePullBackOff" \
  --mode manual \
  --maxRiskLevel low \
  --confidenceThreshold 0.8
```

**Read the output closely — it is a window into how an agent actually works:**

- **`investigation.dataGathered`** lists every step it took. In testing this was a
  9-step investigation: multiple `kubectl_get` and `kubectl_describe` calls, and
  even a **`kubectl_patch_dryrun`** — it validated its proposed change against the
  API server *before* recommending it.
- **`remediation.actions`** contains the concrete fix it recommends — a
  `kubectl patch` changing the image to `nginx:latest` — each action with a
  `rationale` and a `risk` rating.
- **`message`** reports its confidence (e.g. *"identified the root cause with 98%
  confidence"*).
- **`status: awaiting_user_approval`** — because you chose `manual` mode, it stops
  here and waits. **Nothing in your cluster was changed.**

### Why manual mode matters

dot-ai *can* run in `automatic` mode, where it applies fixes itself — but only when
its confidence and the action's risk meet the thresholds you set:

- `--confidenceThreshold 0.8` — only act if at least 80% confident
- `--maxRiskLevel low` — never auto-execute anything riskier than "low"

Manual mode is the safe default for learning, and a sound model for production: you
keep a human in the loop for anything that *mutates* the cluster. This **"investigate
→ validate (dry-run) → propose → gate execution"** pattern is the core discipline of
agentic operations — the AI does the tedious investigation, but a human (or an
explicit threshold) authorizes the change.

### Apply the fix yourself (optional)

If you want to complete the loop, run the `kubectl patch` command dot-ai printed,
then confirm the Pod recovers:

```bash
kubectl set image deployment/broken nginx=nginx:latest
sleep 15
dot-ai query "Is the broken deployment healthy now?"
```

---

## Reflect

- dot-ai treated your request as a *goal* and owned the whole loop, including a
  dry-run before proposing a change. How is that a different relationship with a
  tool than "show me the pods"?
- The platform indexes your cluster's capabilities into a vector database. Why does
  grounding recommendations in *your* cluster's real capabilities beat generic
  advice?
- `manual` mode gated the fix behind your approval. What guardrails (confidence
  thresholds, risk caps, approvals, testing) would *you* require before enabling
  `automatic` mode in production?

### Where this fits

dot-ai sits at the "agentic platform" end of a spectrum of AI Kubernetes tooling:

| Tool | Where the AI runs | Mental model |
|------|-------------------|--------------|
| **kubectl-ai** | Your shell | Autocomplete for cluster commands |
| **kagent** | Inside the cluster, as Pods | A team of specialist agents in your cluster |
| **dot-ai** (this lab) | A server you deploy + a CLI | An AI platform engineer you give goals to |

---

## Cleanup

Stop the port-forward (`Ctrl+C`), then remove dot-ai and the test workloads:

```bash
helm uninstall dot-ai -n dot-ai
kubectl delete namespace dot-ai
kubectl delete deployment nginx broken
rm -f ~/dot-ai-token.txt
```

To reset the cluster completely:

```bash
minikube delete
```

## Lab completion

Congratulations! You have deployed and driven an agentic DevOps platform. Key
takeaways:

1. dot-ai accepts **goals**, not just commands, and runs a multi-step investigation
   grounded in your cluster's actual capabilities (indexed in a vector database).
2. Its CLI is the headless twin of the MCP server an AI assistant would drive — the
   same brain, different client.
3. **Gated remediation** (`manual` mode, confidence and risk thresholds) is the key
   safety pattern: let the AI investigate and propose, but keep a human in the loop
   before anything changes your cluster.
