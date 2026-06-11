# Lab: kagent — AI Agents That Live Inside Your Cluster

> Every command in this lab was tested end-to-end on the class instructor node
> (minikube, Kubernetes v1.35.1) on 2026-06-11 using an OpenAI API key.

## What you will learn

- What `kagent` is and why "agents as Kubernetes resources" is a meaningfully
  different idea from a terminal copilot
- How to install the kagent framework into a cluster
- How agents are defined as Custom Resources, and how to list and invoke them
- How to watch an agent autonomously call tools to investigate a problem
- *Why* you might want AI that is in-cluster, shared, declarative, and auditable

This is one of three labs on AI tooling for Kubernetes. Where the **kubectl-ai**
lab put an AI copilot in *your shell*, this lab moves the intelligence *into the
cluster itself*.

---

## Why kagent?

### The limitation it addresses

A terminal copilot like kubectl-ai is great, but it is fundamentally *personal* and
*ephemeral*: it lives in one engineer's shell, uses that engineer's credentials,
and vanishes when the terminal closes. There is no shared "cluster brain" that your
whole team — or other systems, like an alerting pipeline — can call. And there is
no record of what was asked or done.

`kagent` (a CNCF Sandbox project) takes the opposite approach. It installs a
**framework inside the cluster** that runs AI **agents as Kubernetes resources**.
Each agent is a Custom Resource (`kind: Agent`) backed by its own Pod, with a
defined specialty and a set of tools it is allowed to use.

### Why put agents *in* the cluster?

This is the central idea of the lab. Running agents as cluster resources buys you
properties that a shell tool cannot offer:

- **Shared and always-on.** The agents are a service. Your whole team can call
  them, and so can other systems (CI, alert handlers, chatops bots) — not just
  whoever has a binary installed.
- **Declarative and GitOps-friendly.** An agent is YAML. That means it is
  version-controlled, code-reviewed, and reconciled like every other object in your
  cluster. You can diff an agent, roll it back, and promote it through environments
  exactly as you do a Deployment.
- **Auditable.** Every interaction is recorded (kagent uses a PostgreSQL database
  for session history), so you have a trail of what was asked and what tools ran.
- **Composable and specialized.** kagent ships a *team* of agents — `k8s-agent`,
  `helm-agent`, `istio-agent`, `observability-agent`, `promql-agent`, and more —
  and agents can call other agents and external tool servers (via MCP, the Model
  Context Protocol). This is how you build automation bigger than a single prompt.

### The mental model

If kubectl-ai is "autocomplete in my shell," kagent is **a team of specialist SREs
living in your cluster**, each an addressable, declarative, auditable resource. It
reflects the Kubernetes philosophy that *everything* — including your AI — should
be a reconciled object.

---

## Prerequisites

You need a running single-node Kubernetes cluster and an OpenAI API key.

Start (or restart) minikube with a little extra memory — kagent deploys several
core Pods plus one Pod per agent:

```bash
minikube delete
minikube start --memory=6144
```

Set your OpenAI key in this shell. **kagent reads it from the environment when you
install it**, and creates a default model configuration from it:

```bash
export OPENAI_API_KEY="<your-openai-api-key>"
```

This lab uses `jq` to format the agent's output. It is already installed on the lab
VM, but you can install it if it is ever missing:

```bash
command -v jq >/dev/null || { sudo apt-get update && sudo apt-get install -y jq; }
```

### Create a workload to investigate

Deploy one healthy app and one deliberately broken one so an agent has something
real to diagnose:

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

## Step 1 — Install the kagent CLI

```bash
curl -sL https://raw.githubusercontent.com/kagent-dev/kagent/refs/heads/main/scripts/get-kagent | bash
kagent version
```

The CLI is how you install kagent into the cluster and talk to its agents from the
command line.

---

## Step 2 — Install kagent into the cluster

The `demo` profile installs the controller, supporting services (a PostgreSQL
database for session history, an MCP tool server, and a web UI), and a set of
ready-to-use agents. kagent reads your `OPENAI_API_KEY` and builds a default model
configuration from it.

```bash
kagent install --profile demo
```

This takes a few minutes — it is pulling several container images. Watch the
components come up:

```bash
kubectl get pods -n kagent
```

> **You may briefly see `kagent-controller` in `CrashLoopBackOff`.** This is
> *expected* and is a nice real-world teaching moment: the controller starts before
> the PostgreSQL database is ready and keeps restarting until the database accepts
> connections. Give it a minute and it settles to `Running`. (Startup ordering like
> this is exactly the kind of thing you could later ask an agent to explain.)

---

## Step 3 — See that agents are real Kubernetes objects

This is the "aha" moment of the lab. The agents kagent created are not config in
some external SaaS — they are Custom Resources in your cluster:

```bash
kubectl get agents -n kagent
```

You will see a list like `k8s-agent`, `helm-agent`, `istio-agent`,
`observability-agent`, `promql-agent`, and others. Inspect one as YAML to see that
it is an ordinary, declarative Kubernetes object:

```bash
kubectl get agent k8s-agent -n kagent -o yaml | head -40
```

> **Why this matters:** because an agent is a CRD, everything you already know about
> managing Kubernetes objects applies to your AI. You can store agents in Git,
> review changes in a pull request, apply them with `kubectl apply`, and let the
> kagent controller reconcile them. Your AI capabilities become part of your
> platform's desired state.

---

## Step 4 — Talk to an agent from the CLI

The kagent CLI talks to the in-cluster controller's API on port `8083`. Open a
port-forward in **one terminal** and leave it running:

```bash
kubectl port-forward -n kagent svc/kagent-controller 8083:8083
```

In a **second terminal** (re-export `OPENAI_API_KEY` if needed), first confirm the
CLI can see the agents:

```bash
kagent get agent
```

All agents should show `DEPLOYMENT_READY: true` and `ACCEPTED: true`. Now send the
`k8s-agent` a real troubleshooting question.

kagent always returns a full **structured JSON** record of the agent's work — great
for automation, noisy for a human. To read just the final answer, pipe the output
through `jq` (installed in the Prerequisites) and pull out the text:

```bash
kagent invoke --agent k8s-agent \
  -t "List the pods in the default namespace and tell me which ones are unhealthy and why." \
  | jq -r '.artifacts[].parts[].text'
```

You get a clean, plain-English answer: the `broken-…` pod is in `ImagePullBackOff`
because the image `nginx:doesnotexist` does not exist.

### See the full audit trail

Now run the command **without** the `jq` filter to see what kagent actually returns:

```bash
kagent invoke --agent k8s-agent -t "Which pods are unhealthy and why?"
```

The raw output is a detailed JSON task record. Look through the `history` array and
you will see the agent autonomously decide to call its tools, one after another:

1. `k8s_get_resources` — it lists the pods (and sees `broken-…` is not ready)
2. `k8s_describe_resource` — it describes the failing Pod to read its events

…and then, in a final `text` part, it states the conclusion. (You can also add the
`--stream` flag to watch these steps arrive one event at a time as the agent works —
useful for long investigations, though still JSON.)

> **Why return all that JSON?** kagent is built for automation and auditing, not just
> human chat. The full structured record lets another system consume the result
> programmatically and lets a human audit exactly which tools the agent ran and what
> each returned — which is why you reach for `jq` (or the dashboard, below) when you
> just want the human answer. Under the hood this is the same investigative loop
> kubectl-ai runs — *call a read-only tool, look at the output, decide what to do
> next* — but here it runs inside a shared, recorded, in-cluster service.

---

## Step 5 (optional) — The web UI

For a human-friendly experience, kagent ships a web UI. There is a `kagent dashboard`
command, but it is built for desktop installs where it can open a browser for you — on a
headless lab VM it just prints the manual command and exits. So forward the UI service's
port yourself. On the VM, in a new terminal, run (leave it running):

```bash
kubectl port-forward -n kagent service/kagent-ui 8082:8080
```

The UI now answers on `localhost:8082` **on the VM**. Because the VM has no browser, you
forward that port again to the computer you are connecting from, then open the UI in that
computer's browser.

**Mac/Linux** — in another terminal, use the same SSH command from the Setup lab, adding
`-L` to forward the port:

```bash
ssh -L 8082:localhost:8082 -i lab.pem ubuntu@<VM-IP>
```

**Windows (PuTTY)** — open the PuTTY session you configured in the Setup lab and add a
tunnel before connecting: go to **Connection → SSH → Tunnels**, set **Source port** to
`8082` and **Destination** to `localhost:8082`, click **Add**, then open the session.

With both the `kubectl port-forward` (on the VM) and the SSH tunnel (to your computer)
running, browse to `http://localhost:8082`.

In the UI you can chat with any agent and watch its tool calls stream live — a much
gentler way to *see* the investigation loop than reading raw JSON.

---

## Reflect

- kagent gave agents in-cluster credentials (a ServiceAccount and RBAC). What are
  the security implications, and why would you scope an agent's tools and
  permissions tightly?
- An agent is a CRD. How does that change the way a platform team would manage,
  review, and roll out AI capabilities compared to handing everyone a CLI?
- You invoked one agent for one question. How might the `helm-agent`,
  `observability-agent`, and `promql-agent` combine to handle a more complex
  incident?

### Where this fits

kagent sits in the middle of a spectrum of AI Kubernetes tooling:

| Tool | Where the AI runs | Mental model |
|------|-------------------|--------------|
| **kubectl-ai** | Your shell | Autocomplete for cluster commands |
| **kagent** (this lab) | Inside the cluster, as Pods | A team of specialist agents in your cluster |
| **dot-ai** | A server you deploy + a CLI | An AI platform engineer you give goals to |

---

## Cleanup

Stop the port-forward (`Ctrl+C`), then remove kagent and the test workloads:

```bash
kagent uninstall
kubectl delete deployment nginx broken
```

To reset the cluster completely for the next lab:

```bash
minikube delete
```

## Lab completion

Congratulations! You have deployed a Kubernetes-native AI agent framework and
delegated a troubleshooting task to an in-cluster agent. Key takeaways:

1. kagent runs AI **agents as Kubernetes resources** — shared, always-on,
   declarative, and auditable, rather than a personal shell tool.
2. Each agent autonomously calls read-only tools (`k8s_get_resources`,
   `k8s_describe_resource`, …) to investigate, and records every step.
3. Treating agents as CRDs lets you manage your AI capabilities with the same
   GitOps discipline as the rest of your platform.
