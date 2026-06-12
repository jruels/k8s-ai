### Holmes Operator: Continuous, Kubernetes-native Investigations

> **⚠️ Alpha software:** The Holmes Operator is in **alpha** and is subject to
> breaking changes. CRD fields, behavior, and chart values may change between
> releases. Use it for learning and experimentation, not production gating, and
> pin your chart version if you build anything on top of it.

#### Introduction

In the previous HolmesGPT lab you drove every investigation by hand — you noticed a
problem, opened a terminal, and ran `holmes ask` or `holmes investigate`. That works,
but it has one big limitation: **a human has to be in the loop to start the
investigation.** If nobody is watching at 2 a.m., nobody asks Holmes anything.

The **Holmes Operator** closes that gap. It runs inside your cluster as a controller and
lets you describe health checks *declaratively* as Kubernetes resources. Holmes then runs
those investigations on a schedule, 24/7, and records the results back onto the resource —
the same way a `Job` or `CronJob` works, but the "work" is an AI investigation.

##### **Why an operator? The benefits of the operator pattern for Holmes**

- **Proactive instead of reactive.** Scheduled checks let Holmes spot problems *before*
  a customer or an alert does. You no longer have to be at a keyboard to start an
  investigation.
- **Declarative and GitOps-friendly.** Your investigations become YAML manifests
  (`HealthCheck` / `ScheduledHealthCheck`) that live in Git, get code-reviewed, and are
  applied with `kubectl apply` — exactly like the rest of your cluster config.
- **Kubernetes-native UX.** Checks are first-class API objects. You manage them with the
  tools you already know: `kubectl get`, `describe`, `patch`, `label`, RBAC, namespaces,
  and `-o jsonpath`. No new control plane to learn.
- **Built-in audit history.** Each run's verdict, rationale, and timing are stored in the
  resource's `status`. A `ScheduledHealthCheck` keeps a rolling history of recent runs, so
  you get a queryable record of cluster health over time.
- **Separation of scheduling from execution.** The operator is a lightweight controller
  that delegates the actual investigation to the stateless Holmes API server over HTTP.
  That means the API servers can scale horizontally while scheduling stays in one place.
- **Standardized, reusable runbooks.** A check's `query` is a plain-English runbook. Once
  written, it runs identically every time and for every team member — no copy-pasting CLI
  commands.

> **💲 Cost note:** **Every health check triggers at least one LLM call.** A
> `ScheduledHealthCheck` that runs every minute is 1,440 LLM calls per day. Schedule
> checks conservatively (and with sensible `timeout`s) to keep token costs under control.

##### **Lab objectives**

- Enable the Holmes Operator on an existing HolmesGPT install
- Verify the operator deployment and its CRDs
- Run a one-time investigation with a `HealthCheck`
- Run recurring investigations with a `ScheduledHealthCheck` and read its history
- Manage schedules (enable/disable/update) with `kubectl patch`

#### Prerequisites

This lab **builds directly on the HolmesGPT lab**. Before you start you should already
have, from that lab and on the same VM:

- A running `minikube` cluster with the `metrics-server` addon enabled
- The `robusta` Helm repo added
- An existing `holmes` Helm release deployed (`helm list` shows it)
- The `holmes-secret` secret holding your OpenAI API key
- Your `holmes-values.yaml` file in the current directory

Confirm the Holmes release is present:
```bash
helm list
```

You should see the `holmes` release with status `deployed`. If you don't, go back and
complete **Part 1** of the HolmesGPT lab first.

> The operator does **not** need the `holmes` CLI. Everything in this lab is done with
> `kubectl` and `helm` against resources in the cluster.

### Part 1: Enabling the Operator

When you installed HolmesGPT with Helm, the chart already registered the operator's CRDs.
You can see them even though the operator itself isn't running yet:

```bash
kubectl get crd | grep holmesgpt.dev
```

```
healthchecks.holmesgpt.dev            ...
scheduledhealthchecks.holmesgpt.dev   ...
```

The operator *deployment* is off by default (`operator.enabled: false`). We turn it on by
adding an `operator` block to the values file you already created and running a Helm
upgrade.

##### **Add the operator block to your values file**

Append the operator configuration to `holmes-values.yaml`. The full file should look like
this (the top half is unchanged from the HolmesGPT lab):

```bash
cat <<EOF > holmes-values.yaml
additionalEnvVars:
- name: MODEL
  value: gpt-4.1-mini
- name: OPENAI_API_KEY
  valueFrom:
    secretKeyRef:
      name: holmes-secret
      key: openai-key
operator:
  enabled: true
EOF
```

> By default the operator connects to the in-cluster Holmes API at
> `http://<release-name>-holmes:80` — for our `holmes` release that resolves to
> `http://holmes-holmes:80`, so no extra configuration is needed. You can override it with
> `operator.holmesApiUrl` if your release name differs.

##### **Upgrade the release**

```bash
helm repo update
```

```bash
helm upgrade holmes robusta/holmes -f holmes-values.yaml
```

You should see the release move to `REVISION: 2` with status `deployed`.

#### Verify the Operator

Wait for the operator deployment to become available:
```bash
kubectl wait --for=condition=available --timeout=120s deployment/holmes-operator
```

Check the operator pod is running:
```bash
kubectl get pods -l app.kubernetes.io/name=holmes-operator
```

```
NAME                             READY   STATUS    RESTARTS   AGE
holmes-operator-...              1/1     Running   0          ...
```

Look at the operator logs — you should see it load its config, connect to the Holmes API,
and start its scheduler:
```bash
kubectl logs -l app.kubernetes.io/name=holmes-operator --tail=20
```

```
[INFO] Starting Holmes Operator...
[INFO] Loaded configuration: Holmes API URL=http://holmes-holmes:80, Log Level=INFO
[INFO] Initialized Holmes API client for http://holmes-holmes:80
[INFO] Starting scheduler...
[INFO] Scheduler started successfully
[INFO] Holmes Operator started successfully
```

The operator is now watching the cluster for `HealthCheck` and `ScheduledHealthCheck`
resources.

### Part 2: One-time investigations with HealthCheck

A `HealthCheck` is the operator equivalent of a Kubernetes `Job`: it runs **once**, as
soon as you create it. This is the simplest way to ask Holmes a question declaratively.

##### **Create a HealthCheck**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: holmesgpt.dev/v1alpha1
kind: HealthCheck
metadata:
  name: example-check
  namespace: default
spec:
  query: "Is the default namespace healthy? Check pod status, recent restarts, and warning events."
  timeout: 60
EOF
```

The `query` is your plain-English runbook; `timeout` is the per-check budget in seconds
(default `30`, range `1–300`).

##### **Watch it run**

List health checks (short name `hc`). It starts in the `Running` phase:
```bash
kubectl get hc
```

```
NAME            PHASE     RESULT   AGE
example-check   Running            2s
```

Re-run the command after a few seconds and the phase moves to `Completed` with a
`RESULT` of `pass` or `fail`:
```bash
kubectl get hc
```

```
NAME            PHASE       RESULT   AGE
example-check   Completed   pass     12s
```

##### **Read the verdict and rationale**

The full result — including Holmes' natural-language rationale — is stored in the
resource's `status`. View it with `describe`:
```bash
kubectl describe hc example-check
```

Look at the `Status` section. You'll see fields such as:

```
Status:
  Phase:       Completed
  Result:      pass
  Model Used:  gpt-4.1-mini
  Duration:    7.6
  Message:     Check passed. There are no pods in the default namespace ...
  Rationale:   There are no pods currently found in the default namespace, so no pod
               status or restarts can be evaluated. However, recent events are all of
               type 'Normal' with no warnings ... the namespace is considered healthy.
```

You can also pull individual fields directly — handy for scripting or CI:
```bash
kubectl get hc example-check -o jsonpath='{.status.result}'
```

### Part 3: Recurring investigations with ScheduledHealthCheck

A `ScheduledHealthCheck` is the operator equivalent of a `CronJob`: it runs a `HealthCheck`
on a recurring **cron schedule** and keeps a rolling history of the results. This is where
the "24/7 proactive monitoring" benefit actually shows up.

> **Note on cron and timezone:** schedules use standard 5-field cron syntax and run in
> **UTC**.
> ```
> * * * * *
> │ │ │ │ └─ day of week (0-6, Sun-Sat)
> │ │ │ └─── month (1-12)
> │ │ └───── day of month (1-31)
> │ └─────── hour (0-23)
> └───────── minute (0-59)
> ```

##### **Create a frequent check (for the lab)**

So we don't have to wait an hour to see results, we'll run every minute. **In the real
world you would not schedule this aggressively** (remember: every run is at least one LLM
call) — an hourly check `"0 * * * *"` is far more typical.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: holmesgpt.dev/v1alpha1
kind: ScheduledHealthCheck
metadata:
  name: frequent-pod-check
  namespace: default
spec:
  schedule: "*/1 * * * *"
  query: "Is the default namespace healthy? Check pod status, recent restarts, and warning events."
  timeout: 60
EOF
```

List scheduled checks (short name `shc`):
```bash
kubectl get shc
```

```
NAME                 SCHEDULE      ENABLED   LAST RUN   LAST RESULT   AGE
frequent-pod-check   */1 * * * *   true                               1s
```

Confirm the operator registered the schedule:
```bash
kubectl logs -l app.kubernetes.io/name=holmes-operator --tail=5
```

```
[INFO] Registering schedule for default/frequent-pod-check
[INFO] Added job "ScheduledHealthCheck default/frequent-pod-check" to job store
[INFO] Registered schedule for default/frequent-pod-check with cron: */1 * * * *
```

##### **Watch scheduled runs accumulate**

Wait about 60–90 seconds for the schedule to fire (it triggers at the top of each minute),
then look again. The `LAST RUN` and `LAST RESULT` columns fill in:
```bash
kubectl get shc frequent-pod-check
```

```
NAME                 SCHEDULE      ENABLED   LAST RUN   LAST RESULT   AGE
frequent-pod-check   */1 * * * *   true      25s        pass          90s
```

Each scheduled run **creates its own `HealthCheck` resource** (named with a timestamp
suffix) so you keep a full audit trail. List them:
```bash
kubectl get hc
```

```
NAME                                        PHASE       RESULT   AGE
frequent-pod-check-20260612-151800-1bc14a   Completed   pass     86s
frequent-pod-check-20260612-151900-43dbd0   Completed   fail     26s
```

> **Holmes catching a real problem:** if you still have the broken `bad-image` deployment
> (or the over-provisioned `resource-issues-pod`) from the HolmesGPT lab running, you'll
> see a run flip to `fail` — Holmes detects the `ImagePullBackOff` and the unschedulable
> pod on its own, without anyone asking. That is exactly the proactive monitoring the
> operator is built for. (If your default namespace is clean, every run will `pass`.)

##### **Read the run history**

The scheduled check stores recent runs in `status.history`. View the full status:
```bash
kubectl describe shc frequent-pod-check
```

Or pull just the latest result and the history array with `jsonpath`:
```bash
kubectl get shc frequent-pod-check -o jsonpath='{.status.lastResult}'
```

```bash
kubectl get shc frequent-pod-check -o jsonpath='{.status.history}'
```

Each history entry records the generated check name, the `pass`/`fail` result, the
execution time, the duration, and Holmes' message — a queryable record of cluster health
over time.

#### Managing a schedule

Because these are ordinary Kubernetes objects, you manage them with `kubectl patch` — no
special tooling.

Temporarily **disable** a schedule without deleting it:
```bash
kubectl patch shc frequent-pod-check --type=merge -p '{"spec":{"enabled":false}}'
```

Confirm the operator removed the job from its scheduler:
```bash
kubectl logs -l app.kubernetes.io/name=holmes-operator --tail=3
```

```
[INFO] Removed job default/frequent-pod-check
[INFO] Removed schedule for default/frequent-pod-check
```

**Re-enable** it:
```bash
kubectl patch shc frequent-pod-check --type=merge -p '{"spec":{"enabled":true}}'
```

**Change the cron schedule** (e.g. to every two hours):
```bash
kubectl patch shc frequent-pod-check --type=merge -p '{"spec":{"schedule":"0 */2 * * *"}}'
```

#### Clean up

The every-minute check keeps spending LLM calls, so remove it when you're done. Deleting
the `ScheduledHealthCheck` also cleans up the `HealthCheck` resources it generated:
```bash
kubectl delete shc frequent-pod-check
```

```bash
kubectl delete hc example-check
```

(Optional) If you want to turn the operator back off, set `operator.enabled: false` in
`holmes-values.yaml` and run `helm upgrade holmes robusta/holmes -f holmes-values.yaml`
again.

### Challenge Tasks

1. **Write a targeted scheduled check.** Create a `ScheduledHealthCheck` that runs every
   15 minutes (`*/15 * * * *`) and asks Holmes to specifically check for crash-looping
   pods and image-pull failures across **all** namespaces. Deploy a broken workload
   (`kubectl create deployment bad-image --image=nginx:nonexistent`) and confirm the next
   run reports `fail`.

2. **Use the audit trail.** After several scheduled runs, use `kubectl get shc <name> -o
   jsonpath` to extract just the `executionTime` and `result` of each history entry and
   build a quick timeline of when the namespace was healthy vs. not.

3. **Right-size the cost.** Compare an every-minute schedule to an hourly one and calculate
   the daily LLM-call count for each. Decide what cadence you'd actually run in production
   and explain why.

### Lab Completion

Congratulations! You enabled and used the Holmes Operator. You learned:

- **Why** the operator exists — it makes investigations *proactive* and *declarative*
  instead of human-triggered and ad-hoc
- The **benefits of the operator pattern**: GitOps-friendly YAML, Kubernetes-native
  management, built-in audit history, and separation of scheduling from execution
- How to enable the operator with a Helm upgrade and verify its deployment and CRDs
- How to run one-time investigations with `HealthCheck` and recurring ones with
  `ScheduledHealthCheck`, including reading verdicts, rationale, and history
- How to manage schedules with `kubectl patch`

Key takeaways:
1. The operator turns HolmesGPT into a 24/7 background investigator
2. `HealthCheck` ≈ `Job` (run once); `ScheduledHealthCheck` ≈ `CronJob` (run on a cron)
3. Results, rationale, and history live in the resource `status` — fully queryable
4. **Every run is at least one LLM call** — schedule checks with cost in mind
5. It's **alpha** software — expect breaking changes and pin your versions
