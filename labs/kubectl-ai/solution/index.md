# Lab: kubectl-ai — An AI Copilot in Your Terminal



## What you will learn

- What `kubectl-ai` is and the specific problem it solves
- How to install and point it at the OpenAI backend
- How to use it for one-shot questions, interactive sessions, and as a `kubectl` plugin
- *Why* it shows you the command before running it — and what that tells you about
  how these AI tools actually work

This is the first of three labs on AI tooling for Kubernetes. It is the lightest
of the three on purpose: it introduces the core idea — *letting an LLM drive
`kubectl` for you* — with nothing to install in the cluster.

---

## Why kubectl-ai?

### The problem it solves

Operating Kubernetes means remembering a *lot* of `kubectl` syntax: the right
subcommand, the resource type, the namespace flag, the `-o jsonpath=…` incantation
to pull one field out of a Pod, the `--sort-by` to order events. Even experienced
engineers keep a cheat sheet. When you are mid-incident, that friction is
expensive — every second spent recalling flag syntax is a second not spent
thinking about the actual problem.

`kubectl-ai` (an open-source tool from Google Cloud) removes that friction. You
describe what you want in plain English; it figures out the exact `kubectl`
command, shows it to you, runs it, and explains the result.

### Where the intelligence lives

This is the key design choice that makes kubectl-ai distinct: **everything runs in
your shell.** There is no operator, no Pod, no Custom Resource Definition installed
in the cluster. The tool reads your existing kubeconfig — so it has exactly the
permissions *you* already have, nothing more — and talks directly to an LLM
provider.

That gives it two defining characteristics:

- **Zero footprint, instant adoption.** You can use it against any cluster you can
  already reach, today, without asking a platform team to install anything or
  granting an agent in-cluster credentials.
- **It lives and dies with your terminal.** There is no shared, always-on
  intelligence — it is *your* personal copilot, not a team service. (That is the
  gap the kagent and dot-ai labs explore.)

### The mental model

Think of kubectl-ai as **autocomplete for cluster operations**. You bring the
intent and the judgment; it handles the translation to syntax and the first pass at
interpreting the output. Crucially — as you will see — it always shows you the
command it is about to run, so you stay in control and keep learning the real
`kubectl` underneath.

---

## Prerequisites

You need a running single-node Kubernetes cluster and an OpenAI API key. If you
have just finished another lab, start from a clean cluster so the output matches
this guide.

Start (or restart) minikube:

```bash
minikube delete
minikube start --memory=5120
```

Set your OpenAI key in this shell. **kubectl-ai reads it from the environment, so
keep this terminal open** (re-export it if you reconnect over SSH):

```bash
export OPENAI_API_KEY="<your-openai-api-key>"
```

### Create a workload to investigate

So the tool has something real to find, deploy one healthy app and one deliberately
broken one:

```bash
# Healthy
kubectl create deployment nginx --image=nginx

# Broken on purpose: the image tag does not exist
kubectl create deployment broken --image=nginx:doesnotexist
```

Give Kubernetes a few seconds, then confirm the broken Pod is stuck in
`ImagePullBackOff`:

```bash
kubectl get pods
```

```
NAME                      READY   STATUS             RESTARTS   AGE
broken-7b58495bf7-xxxxx   0/1     ImagePullBackOff   0          20s
nginx-56c45fd5ff-xxxxx    1/1     Running            0          20s
```

This is the problem you will ask kubectl-ai to diagnose.

---

## Step 1 — Install kubectl-ai

The install script downloads the right binary for your platform and places it on
your `PATH`:

```bash
curl -sSL https://raw.githubusercontent.com/GoogleCloudPlatform/kubectl-ai/main/install.sh | bash
```

Verify it:

```bash
kubectl-ai version
```

**A note on providers.** kubectl-ai defaults to Google Gemini. Because this class
uses an OpenAI key, the very next step points kubectl-ai at OpenAI — *once*, in a
config file — so you don't have to repeat provider/model flags on every command.

---

## Step 2 — Configure the provider and model (once)

kubectl-ai reads a config file at `~/.config/kubectl-ai/config.yaml`. Setting your
provider and model there means every later command is just `kubectl-ai "..."` — no
flags to retype. Create it now:

```bash
mkdir -p ~/.config/kubectl-ai
cat > ~/.config/kubectl-ai/config.yaml <<EOF
llmProvider: openai
model: gpt-4.1
EOF
```

Confirm kubectl-ai picks it up — this query passes **no** provider/model flags, yet it
will answer using OpenAI:

```bash
kubectl-ai --quiet "what deployments are in the default namespace?"
```

**Config vs. flags — you can use either.** The config file sets your *defaults*, which
is why the rest of this lab uses plain `kubectl-ai "..."`. You can still pass
`--llm-provider` and `--model` on any command for a one-off, and a flag always wins
over the config file — e.g. `kubectl-ai --model=gpt-4.1-mini --quiet "..."` uses a
cheaper model just for that call. Two things the config file does **not** hold: your
**API key** (that stays in the `OPENAI_API_KEY` environment variable you exported) and
your kubeconfig (kubectl-ai uses your normal `kubectl` context).

---

## Step 3 — Ask a one-shot question

The `--quiet` flag runs a single request and exits — the simplest way to try the
tool. Ask it to list the pods:

```bash
kubectl-ai --quiet \
  "list all pods in the default namespace and their status"
```

**Read the output carefully.** kubectl-ai prints the command it chose:

```
  Running: kubectl get pods -n default -o wide
```

…and then summarizes the result in English, calling out that `broken-…` is in
`ErrImagePull`/`ImagePullBackOff` while `nginx-…` is `Running`.

**Why does it show you the command?** This is the single most important habit to
notice. The AI is not a black box that "just knows" your cluster — it decided that
the way to answer your question was to run `kubectl get pods`, exactly as you
would have. By surfacing that command, kubectl-ai keeps you in the loop, lets you
sanity-check its reasoning, and quietly teaches you the `kubectl` you might not
have known. **Never trust output you cannot trace back to a command.**

---

## Step 4 — Ask it to *diagnose*, not just list

Listing status is easy. The real value shows up when you ask an open-ended question
that requires the tool to gather more information and reason about it:

```bash
kubectl-ai --quiet \
  "why is the broken deployment failing, and what is the exact fix?"
```

This time kubectl-ai will run `kubectl describe` on the failing Pod, read the
events, and explain that the image tag `nginx:doesnotexist` does not exist in the
registry — and recommend correcting it to a valid tag such as `nginx:latest`.

Notice what just happened: the tool ran **a different command than before**,
because the *question* was different. It chose `describe` (which surfaces events)
over `get` (which only shows status). That choice — picking the right
investigative command for the question — is exactly the skill a junior engineer
spends months learning, and it is what the AI is automating.

### Try a few more

Each of these makes kubectl-ai pick a different command. Predict which `kubectl`
command it will choose *before* you run it, then check whether you were right:

```bash
kubectl-ai --quiet \
  "show me the most recent events in the default namespace"

kubectl-ai --quiet \
  "which container image is the broken deployment trying to use?"

kubectl-ai --quiet \
  "how many deployments are in the default namespace and how many replicas does each want?"
```

---

## Step 5 — Interactive mode

Launched with **no query**, kubectl-ai opens a conversation you can hold across
multiple turns — useful when one answer leads to a follow-up question and you want
the tool to remember the context:

```bash
kubectl-ai
```

Try a sequence like:

```
> what pods are unhealthy?
> describe the unhealthy one
> what's the smallest change that would fix it?
```

Type `exit` (or press `Ctrl+C`) to leave the session.

**Why interactive mode?** Real troubleshooting is a conversation, not a single
question. Interactive mode lets the model build on what it already discovered, so
you do not have to re-state context each time. This is the same multi-turn loop
the other two tools automate more heavily.

---

## Step 6 — Use it as a kubectl plugin

Because the binary is named `kubectl-ai`, the `kubectl` command automatically
exposes it as a subcommand — so it feels like a native part of your existing
workflow:

```bash
kubectl ai --quiet \
  "is the nginx deployment healthy?"
```

This is a small thing, but it matters for adoption: engineers do not have to learn
a new top-level command, they just add `ai` after `kubectl`.

---

## Reflect

- kubectl-ai never created or changed a single object — it only *read* from your
  cluster, using your own permissions. Why does that make it safe to adopt quickly?
- Every answer was backed by a visible `kubectl` command. How does that visibility
  protect you, and how does it help you learn?
- kubectl-ai disappears when you close your terminal. When would you want an
  always-on, shared AI service for your cluster instead? (That is the problem the
  **kagent** lab tackles.)

### Where this fits

kubectl-ai sits at the "copilot" end of a spectrum of AI Kubernetes tooling:

| Tool | Where the AI runs | Mental model |
|------|-------------------|--------------|
| **kubectl-ai** (this lab) | Your shell | Autocomplete for cluster commands |
| **kagent** | Inside the cluster, as Pods | A team of specialist agents in your cluster |
| **dot-ai** | A server you deploy + a CLI | An AI platform engineer you give goals to |

---

## Cleanup

```bash
kubectl delete deployment nginx broken
```

To reset the cluster completely for the next lab:

```bash
minikube delete
```

## Lab completion

Congratulations! You have used an AI copilot to operate Kubernetes from plain
English. Key takeaways:

1. kubectl-ai turns natural language into `kubectl` commands with **zero cluster
   footprint** — it runs entirely in your shell with your own permissions.
2. Set your provider and model **once** in `~/.config/kubectl-ai/config.yaml` instead
   of repeating flags; a CLI flag still overrides the config for one-off calls, and your
   API key stays in the `OPENAI_API_KEY` environment variable.
3. It **always shows the command** it runs, keeping you in control and teaching you
   the underlying `kubectl`.
4. It is a *personal* copilot. When you need shared, always-on, or goal-driven AI
   for a whole team or cluster, you reach for heavier tools — which the next two
   labs introduce.
