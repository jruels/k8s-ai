# Lab: kagent (Agentic) — From Diagnosing to *Fixing*

## What you will learn

In the first kagent lab you used an in-cluster agent to **identify** problems. In
this lab you let agents **resolve** them. You will:

- Have the built-in `k8s-agent` autonomously **remediate** real failures — patch a
  broken image, scale a deployment, repair a misconfigured Service, and create
  brand-new resources from a plain-English goal
- Watch an agent's **safety guardrails** in action, and use **sessions** to approve a
  proposed fix in a follow-up turn
- **Stream** an agent's tool calls so you can see exactly what it does, step by step
- Define your **own remediation agent as a Kubernetes CRD** with a scoped tool set —
  the GitOps way to ship a safe, purpose-built automation
- Reason about **RBAC, blast radius, and least privilege** when you let AI write to
  your cluster

> This lab assumes you have already completed the **kagent** lab: kagent is installed
> with the `demo` profile, the agents show `READY`/`ACCEPTED`, and you understand that
> agents are Kubernetes Custom Resources. If not, do that lab first.

---

## Why "fixing" is a bigger deal than "finding"

A read-only assistant is low-stakes: the worst it can do is be wrong in a sentence. An
agent that can `patch`, `apply`, and `delete` your cluster is a different proposition —
it is taking **actions** with real consequences. That is exactly why kagent's design
matters here:

- **The agent's power is its tool set, and the tool set is declared in YAML.** An agent
  can only do what its tools allow. Scope the tools tightly and you cap the blast radius
  — the agent literally cannot delete a resource if you didn't give it a delete tool.
- **Guardrails are built in.** The default `k8s-agent` is prompted to follow least
  privilege and to **confirm before destructive or significant changes**. You will see it
  stop and ask.
- **Everything is recorded.** Each fix is a session with a full history of which tools ran
  and what they returned — an audit trail for changes the AI made.

The takeaway: autonomous remediation is useful *because* it is bounded, declarative, and
auditable — the same properties that make Kubernetes itself safe to operate.

---

## Prerequisites

You need the kagent install from the previous lab, plus `jq` (already on the lab VM) to
read the agents' JSON output.

Confirm the agents are up:

```bash
kubectl get agents -n kagent
```

You should see `k8s-agent` (and others) with `READY: True` and `ACCEPTED: True`.

### Open a port-forward to the controller

The kagent CLI talks to the in-cluster controller API on port `8083`. Open this in **one
terminal** and leave it running for the whole lab:

```bash
kubectl port-forward -n kagent svc/kagent-controller 8083:8083
```

In a **second terminal**, confirm the CLI can reach the agents:

```bash
kagent get agent
```

> **A note on prompts in this lab.** When you want an agent to *act*, two things make it
> reliable: (1) tell it to **investigate first** so it discovers exact names instead of
> guessing, and (2) explicitly **authorize it to proceed without confirmation** when you
> want a one-shot fix. You will see both patterns below — and in the Sessions section
> you'll see what happens when you *don't* pre-authorize.

---

## Scenario 1 — Self-heal a bad image

Deploy a workload that can never start because its image doesn't exist:

```bash
kubectl create deployment broken --image=nginx:doesnotexist
```

Confirm it's stuck:

```bash
kubectl get deploy broken
```

```
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
broken   0/1     1            0           5s
```

Now hand the problem to the agent and authorize it to fix it:

```bash
kagent invoke --agent k8s-agent \
  -t "Investigate the deployment named broken in the default namespace which is in ImagePullBackOff. First read its YAML to find the exact container name and current image. Then you are authorized, without asking for confirmation, to patch that container image to nginx:latest. After patching, verify the pod reaches Running and report the container name you changed and the final status." \
  | jq -r '.artifacts[].parts[].text'
```

The agent reads the deployment, finds the container name, patches the image, and reports
that the pod is now `Running`. Verify for yourself:

```bash
kubectl get deploy broken -o wide
```

```
NAME     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
broken   1/1     1            1           50s   nginx        nginx:latest   app=broken
```

The agent didn't just *tell you* the image was wrong — it **changed it** and confirmed the
recovery. That is the whole point of this lab.

---

## Scenario 2 — Scale to meet demand

Create a single-replica app:

```bash
kubectl create deployment web --image=nginx
```

Ask the agent to scale it (this exercises the agent's *resource management* skill):

```bash
kagent invoke --agent k8s-agent \
  -t "Scale the deployment named web in the default namespace to 3 replicas. You are authorized to perform this change without asking for confirmation. Then report the new replica count." \
  | jq -r '.artifacts[].parts[].text'
```

Verify the scale-up:

```bash
kubectl get deploy web
```

```
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    3/3     3            3           15s
```

---

## Scenario 3 — Repair a misconfigured Service

A classic, maddening bug: a Service whose selector matches no pods, so it silently routes
to nothing. Create one pointing at a label that doesn't exist:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: default
spec:
  selector:
    app: wrong-label
  ports:
  - port: 80
    targetPort: 80
EOF
```

Confirm it has no endpoints (so it's effectively dead):

```bash
kubectl get endpoints web-svc
```

```
NAME      ENDPOINTS   AGE
web-svc   <none>      1s
```

Let the agent diagnose the selector/label mismatch and fix it:

```bash
kagent invoke --agent k8s-agent \
  -t "The Service web-svc in the default namespace has no endpoints, so it routes to nothing. Investigate by reading the Service selector and the labels on the web deployment pods. The pods are healthy. You are authorized, without asking for confirmation, to patch the Service selector so it correctly matches the web pods. Then verify the Service has endpoints and report the selector you set." \
  | jq -r '.artifacts[].parts[].text'
```

The agent compares the Service selector against the pod labels, patches the selector to
`app: web`, and the endpoints populate:

```bash
kubectl get endpoints web-svc
```

```
NAME      ENDPOINTS                                      AGE
web-svc   10.244.0.27:80,10.244.0.28:80,10.244.0.29:80   22s
```

This is remediation that requires *reasoning across two objects* — the kind of fix that's
tedious by hand and a natural fit for an agent.

---

## Scenario 4 — Create new resources from a goal

Remediation isn't only patching what exists — sometimes the fix is to **create** what's
missing. Ask the agent to stand up a small app from a plain-English description; it will
generate the manifests and apply them:

```bash
kagent invoke --agent k8s-agent \
  -t "In the default namespace, create a new Deployment named cache running the redis:7 image with 1 replica and container port 6379, and a ClusterIP Service named cache that exposes port 6379 targeting those pods. You are authorized to apply these resources without asking for confirmation. Then verify the deployment is available and the service has an endpoint, and report what you created." \
  | jq -r '.artifacts[].parts[].text'
```

Verify the agent created and wired up both objects:

```bash
kubectl get deploy cache
kubectl get svc cache
kubectl get endpoints cache
```

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
cache   1/1     1            1           20s
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
cache   ClusterIP   10.98.224.137   <none>        6379/TCP   20s
NAME    ENDPOINTS          AGE
cache   10.244.0.30:6379   21s
```

The agent translated intent into correct, connected Kubernetes objects — Deployment,
Service, matching labels, and a live endpoint — without you writing a line of YAML.

---

## Scenario 5 — Guardrails and sessions: approve a fix in a follow-up turn

So far you've *pre-authorized* every change. Now see what the agent does when you don't —
and how **sessions** let you approve afterward, the way a human operator would.

First, break the deployment again:

```bash
kubectl set image deployment/broken nginx=nginx:doesnotexist
```

Ask the agent to propose a fix **but require your confirmation**:

```bash
kagent invoke --agent k8s-agent \
  -t "The deployment broken in the default namespace is in ImagePullBackOff. Investigate the root cause and propose a fix, but ask me to confirm before making any change."
```

This time, run it **without** the `jq` filter — you want the full JSON, because you need
the **`contextId`** near the top of the output. The agent will lay out a plan and end by
asking you to confirm; it makes **no change**. Copy the `contextId` value, for example:

```
"contextId":"060f485d-c9cc-4bf6-8a6c-97a57093b812",
```

Now **continue that same conversation** by passing the id to `--session`, and approve:

```bash
kagent invoke --agent k8s-agent --session "<paste-contextId-here>" \
  -t "Yes, proceed with nginx:latest. Apply the patch now and verify the pod is Running." \
  | jq -r '.artifacts[].parts[].text'
```

Because it's the same session, the agent **remembers** the plan it proposed — which
deployment, which image — and now executes it. Verify:

```bash
kubectl get deploy broken -o wide
```

```
NAME     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
broken   1/1     1            1           47s   nginx        nginx:latest   app=broken
```

**Why this matters:** the propose → confirm → execute flow, with memory across turns, is
how you keep a human in the loop for risky changes while still letting the agent do the
work. The session is also your audit record of the exchange.

---

## Scenario 6 — Watch the work with `--stream`

When an agent is *changing* your cluster, "trust me, I fixed it" isn't good enough. The
`--stream` flag emits every step — each tool call and result — as it happens. The raw
stream is JSON, but a small `jq` filter turns it into a readable timeline:

```bash
kagent invoke --stream --agent k8s-agent \
  -t "List the pods in the default namespace and tell me which are unhealthy." \
  | jq -rc 'select(.kind=="status-update") | .status.message.parts[]? |
      if (.kind=="data" and .data.name and (.data | has("args"))) then "TOOL CALL  -> " + .data.name
      elif .kind=="text" then "AGENT      -> " + (.text[0:90])
      else empty end'
```

You'll see the agent narrate its plan, then call its tools, then conclude:

```
AGENT      -> List the pods in the default namespace and tell me which are unhealthy.
TOOL CALL  -> k8s_get_resources
AGENT      -> I have listed the pods in the default namespace. All the pods are currently in the "Runnin
```

For a real remediation you'd see `k8s_describe_resource`, `k8s_patch_resource`, and so on
scroll by — a live, inspectable record of exactly what the agent is doing to your cluster.

---

## Scenario 7 — Build your own remediation agent (as a CRD)

The built-in `k8s-agent` is a generalist with a broad tool set. For production you often
want a **purpose-built agent with a narrow, safe tool set** — and because an agent is just
a Custom Resource, you ship it the same way you ship any Kubernetes object: as reviewed,
version-controlled YAML.

We'll define `remediation-agent`: it can **read, patch, and apply** — but we deliberately
**do not give it a delete tool**. That single omission means the agent *cannot* delete a
resource no matter what anyone asks it. This is least privilege expressed as configuration.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: remediation-agent
  namespace: kagent
spec:
  type: Declarative
  description: A focused agent that diagnoses and repairs unhealthy workloads.
  declarative:
    modelConfig: default-model-config
    runtime: python
    stream: true
    systemMessage: |-
      You are RemediationBot, an SRE agent that not only diagnoses Kubernetes
      problems but fixes them. Workflow for every request:
      1. Investigate first: read the resource YAML, events, and logs to find the
         exact root cause and the precise field that is wrong.
      2. Apply the smallest safe change that fixes it, using patch or apply.
      3. Verify the fix took effect, then report what you changed and why.
      You can read, patch, and apply resources, but you cannot delete them.
      Format responses as Markdown.
    tools:
    - type: McpServer
      mcpServer:
        apiGroup: kagent.dev
        kind: RemoteMCPServer
        name: kagent-tool-server
        namespace: kagent
        toolNames:
        - k8s_get_resources
        - k8s_describe_resource
        - k8s_get_events
        - k8s_get_pod_logs
        - k8s_patch_resource
        - k8s_apply_manifest
EOF
```

The kagent controller reconciles your YAML into a running agent (its own Pod). Wait for it
to become ready:

```bash
kubectl get agent remediation-agent -n kagent
```

```
NAME                TYPE          RUNTIME   READY   ACCEPTED
remediation-agent   Declarative   python    True    True
```

Now test it. Notice how little you have to say — the *agent's own system prompt* already
encodes the investigate → fix → verify workflow and the authorization to patch, so a terse
request is enough. Break the deployment and ask:

```bash
kubectl set image deployment/broken nginx=nginx:doesnotexist
```

```bash
kagent invoke --agent remediation-agent \
  -t "The deployment broken in the default namespace is unhealthy. Fix it." \
  | jq -r '.artifacts[].parts[].text'
```

```bash
kubectl get deploy broken -o wide
```

```
NAME     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
broken   1/1     1            1           31s   nginx        nginx:latest   app=broken
```

You just built a specialized, least-privilege remediation bot — its behavior baked into a
reusable prompt and its powers bounded by a declared tool list. That YAML can live in Git,
be reviewed in a PR, and be promoted across clusters exactly like a Deployment.

---

## Reflect

- **Tools = capabilities.** `remediation-agent` couldn't delete a resource even if asked,
  because no delete tool was in its spec. How would you use this to give different teams
  agents with different, auditable power levels?
- **RBAC is the real backstop.** The agent acts through a ServiceAccount; its Kubernetes
  RBAC is the ultimate limit on blast radius, independent of the prompt. Why is it unwise
  to rely on the system prompt alone to keep an agent "safe"?
- **Human-in-the-loop vs. autonomous.** Scenario 5 (confirm-then-apply) and Scenario 7
  (one-shot fix) are two different operating points. For which kinds of changes would you
  require confirmation, and which would you let run unattended?
- **Auditability.** Every fix was a recorded session. How does an immutable trail of "which
  tool ran, with what arguments, and what it returned" change incident review for changes
  an AI made?

---

## Cleanup

Stop the port-forward (`Ctrl+C` in its terminal), then remove what you created:

```bash
kubectl delete deployment broken web cache --ignore-not-found
kubectl delete svc web-svc cache --ignore-not-found
kubectl delete agent remediation-agent -n kagent --ignore-not-found
```

To reset the cluster completely for the next lab:

```bash
minikube delete
```

## Lab completion

Congratulations! You moved kagent from *diagnosing* to *fixing*. Key takeaways:

1. The built-in agents can **remediate** — patch images, scale, repair Services, and
   create resources — not just report problems.
2. **Sessions** give agents memory across turns, enabling a propose → confirm → execute
   workflow that keeps a human in the loop for risky changes.
3. **`--stream`** exposes every tool call, so an agent's actions on your cluster are
   inspectable in real time.
4. An agent's power is defined by its **declared tool set**; scoping tools (e.g. omitting
   delete) is least privilege as configuration, and **RBAC** is the ultimate backstop.
5. Because an agent is a **CRD**, a purpose-built remediation bot is shipped, reviewed, and
   rolled out with the same GitOps discipline as the rest of your platform.
