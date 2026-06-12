### Part 1: Installing HolmesGPT with Helm


#### Prerequisites

Start minikube with 6GB memory:
```bash
minikube start --memory=5120
```

If minikube is already running, you can update its resources:
```bash
minikube stop
minikube delete
minikube start --memory=5120
```

Note: If you have resources deployed from a previous lab, run `minikube delete` to start with a fresh cluster.

Install the metrics API

```bash
minikube addons enable metrics-server
```

#### Introduction

##### **Overview of the Lab Objectives**
- Install HolmesGPT
- Configure different LLM backends
- Test different investigation scenarios
- Understand security and configuration options

##### **Brief on HolmesGPT**
- **HolmesGPT**: An AI-powered tool for investigating and resolving Kubernetes alerts
- **Key Features**:
  - Automatic data collection
  - Read-only access with RBAC permissions
  - Runbook automation
  - Multiple LLM backend support
  - Integration with existing tools (PagerDuty, etc.)

#### Environment Setup

##### **Set your OpenAI API key**


```bash
export OPENAI_TOKEN="<your-openai-api-key>"
```

Verify it is set (you should see your key, not a blank line):
```bash
echo "$OPENAI_TOKEN"
```


##### **Create Configuration File**

Create a values file for Helm configuration:
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
EOF
```

##### **Create Secret for API Key**
```bash
kubectl create secret generic holmes-secret \
  --from-literal=openai-key=$OPENAI_TOKEN
```

##### **Add Helm Repository**
```bash
helm repo add robusta https://robusta-charts.storage.googleapis.com
```

```bash
helm repo update
```

##### **Install HolmesGPT**
```bash
helm install holmes robusta/holmes -f holmes-values.yaml
```

### Part 2: Testing HolmesGPT

#### Create Test Scenarios

##### **Deploy a Pod with Resource Issues**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-issues-pod
  labels:
    app: resource-issues
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "1000Gi"  # Intentionally excessive
        cpu: "100"        # Intentionally excessive
EOF
```

##### **Deploy a Pod with Image Issues**
```bash
kubectl create deployment bad-image --image=nginx:nonexistent
```

#### Using HolmesGPT

##### **Install HolmesGPT CLI**

First, install pipx if not already installed:
```bash
sudo apt update
sudo apt install -y pipx
```

Install HolmesGPT using pipx:
```bash
pipx install holmesgpt
```

Update your PATH:
```bash
pipx ensurepath
source ~/.bashrc  # or restart your terminal
```


##### **Configure HolmesGPT**

You can either:

1. Use the config file:
   - Note that you do need the double quotes around your API key here
   - Make sure `OPENAI_TOKEN` is still exported in this shell (see the first step of the lab)

```bash
mkdir -p ~/.holmes
cat <<EOF > ~/.holmes/config.yaml
model: "gpt-4.1-mini"
api_key: "$OPENAI_TOKEN"
EOF
```

2. Or use the command line flag:
```bash
holmes ask --api-key=$OPENAI_TOKEN "what pods are unhealthy in my cluster?"
```

##### **Basic Usage**

Ask about cluster issues:
```bash
holmes ask "what pods are unhealthy in my cluster?"
```

Check exposed services:
```bash
holmes ask "what services does my cluster expose externally?"
```

Investigate specific resources:
```bash
holmes ask "why is my resource-issues-pod pending?"
```

##### **Advanced Usage Examples**

Investigate specific namespaces:
```bash
holmes ask "what's happening in the kube-system namespace?"
```

Get detailed pod analysis:
```bash
holmes ask "analyze the resource usage of all pods in my cluster"
```

Debug networking issues:
```bash
holmes ask "are there any networking issues between my services?"
```


###### **Scenario 1: Fixing Resource Configuration Issues**


Now, let's use HolmesGPT to analyze and fix the issues:
```bash
# Ask HolmesGPT to analyze the resource configuration
holmes ask "analyze the resource configuration of pod resource-issues-pod and provide a fixed YAML. ONLY PROVIDE USER-CONFIGURABLE FIELDS"
```

HolmesGPT will suggest just the essential configuration:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: resource-issues
  name: resource-issues-pod
  namespace: default
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      requests:
        cpu: "1"
        memory: "1Gi"
  restartPolicy: Always
```


Apply the fixed YAML suggested by HolmesGPT:
```bash
kubectl delete pod resource-issues-pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-issues-pod
  labels:
    app: resource-issues
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "512Mi"  # Reduced memory
        cpu: "100m"      # Reduced CPU
EOF
```

###### **Scenario 2: Fixing Excessive Memory Usage**

Deploy a pod with excessive memory usage:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-heavy-pod
  labels:
    app: memory-test
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "8Gi"
      limits:
        memory: "16Gi"
EOF
```

Ask HolmesGPT to analyze the pod and suggest appropriate resource limits:
```bash
holmes ask "analyze memory-heavy-pod and suggest a complete YAML manifest with appropriate memory limits based on nginx best practices. ONLY PROVIDE USER-CONFIGURABLE FIELDS"
```

Apply the fixed YAML suggested by HolmesGPT:
```bash
kubectl delete pod memory-heavy-pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-heavy-pod
  labels:
    app: memory-test
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
EOF
```

Verify the fix:
```bash
holmes ask "verify if memory-heavy-pod is now running with appropriate resource limits"
```

###### **Scenario 3: Fixing Liveness Probe Issues**

Deploy a pod with problematic liveness probe:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-issues-pod
spec:
  containers:
  - name: nginx
    image: nginx
    livenessProbe:
      httpGet:
        path: /nonexistent
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 3
EOF
```

Get HolmesGPT's analysis and fix:
```bash
# Ask for probe configuration analysis and fix
holmes ask "analyze the liveness probe configuration of probe-issues-pod and provide a fixed YAML. ONLY PROVIDE USER-CONFIGURABLE FIELDS"
```

Apply the suggested fixed configuration:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-issues-pod
spec:
  containers:
  - name: nginx
    image: nginx
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
EOF
```

###### **Scenario 4: Fixing Service Endpoint Issues**

Deploy a service with missing endpoints:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: missing-endpoint-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nonexistent
EOF
```

Get HolmesGPT's analysis and fix:
```bash
# Ask for service configuration analysis
holmes ask "analyze the service missing-endpoint-svc and its endpoints, then provide a fixed YAML including the necessary deployment. ONLY PROVIDE USER-CONFIGURABLE FIELDS"
```

Apply the suggested fixed configuration:
```bash
# First create the deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Then create the properly configured service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: missing-endpoint-svc
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: backend
EOF
```

###### **Scenario 5: Fixing Security Context Issues**

Deploy a pod with security issues:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: security-issues-pod
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
      runAsUser: 0
EOF
```

Get HolmesGPT's analysis and fix:
```bash
# Ask for security configuration analysis
holmes ask "analyze the security context of security-issues-pod and provide a fixed YAML following security best practices. ONLY PROVIDE USER-CONFIGURABLE FIELDS"
```

Apply the suggested fixed configuration:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: security-issues-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
    resources:
      limits:
        memory: "256Mi"
        cpu: "500m"
      requests:
        memory: "128Mi"
        cpu: "250m"
EOF
```

###### **Scenario 6: Fixing Pod Affinity Issues**

Deploy pods with problematic affinity rules:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: affinity-issue-pod
spec:
  containers:
  - name: nginx
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nonexistent-label
            operator: In
            values:
            - nonexistent-value
EOF
```

Get HolmesGPT's analysis and fix:
```bash
# Ask for affinity configuration analysis
holmes ask "analyze the affinity rules of affinity-issue-pod and provide a fixed YAML. ONLY PROVIDE USER-CONFIGURABLE FIELDS"
```

HolmesGPT will suggest just the essential configuration:
```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
  containers:
  - name: nginx
    image: nginx
```

###### **Scenario 7: Fixing Network Policy Issues**

Deploy a problematic network policy:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrictive-policy
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
EOF
```

Get HolmesGPT's analysis and fix:
```bash
# Ask for network policy analysis
holmes ask "analyze the network policy restrictive-policy and provide a fixed YAML that allows essential traffic. ONLY PROVIDE USER-CONFIGURABLE FIELDS"
```

HolmesGPT will suggest just the essential configuration:
```yaml
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

For each scenario above, you can also use HolmesGPT to verify the fixes:
```bash
# Verify the fix
holmes ask "verify if the recent changes to [pod/service name] resolved the issues"

# Get a comprehensive analysis
holmes ask "analyze the current state of the cluster and confirm all issues are resolved"
```

### Part 3: Custom Runbooks with AlertManager

First, let's set up Prometheus and our test application:

#### Install Prometheus Stack
```bash
# Clone the kube-prometheus repository
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus

# Create the namespace and CRDs
kubectl create -f manifests/setup

# Create the monitoring stack
kubectl create -f manifests/
```

---

Expected, harmless errors: Because you enabled the `metrics-server` addon in the
prereqs, it already owns the `v1beta1.metrics.k8s.io` APIService. When you run
`kubectl create -f manifests/`, the bundled `prometheus-adapter` tries to claim the same
APIService and you will see two `AlreadyExists` errors:

```
Error from server (AlreadyExists): ... apiservices ... "v1beta1.metrics.k8s.io" already exists
Error from server (AlreadyExists): ... clusterroles ... "system:aggregated-metrics-reader" already exists
```

These are safe to ignore — Prometheus and AlertManager (which is all this part of the lab
needs) still install and run correctly. You can confirm with
`kubectl get pods -n monitoring`.

---

#### Deploy Test Application
```bash
# Deploy a web app with Prometheus metrics
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "80"
    spec:
      containers:
      - name: webapp
        image: nginx/nginx-prometheus-exporter:0.10.0
        ports:
        - containerPort: 80
EOF
```

#### Create AlertManager Rules
```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: webapp-alerts
  namespace: monitoring
spec:
  groups:
  - name: webapp
    rules:
    - alert: WebAppDown
      expr: up{app="webapp"} == 0
      for: 1m
      labels:
        severity: critical
    - alert: WebAppSlow
      expr: nginx_http_request_duration_seconds_sum{app="webapp"} > 1
      for: 5m
      labels:
        severity: warning
EOF
```

#### Investigate Alerts with HolmesGPT

First, set up port forwarding for AlertManager:
```bash
kubectl port-forward -n monitoring svc/alertmanager-main 9093:9093
```

**Note:** AlertManager is created by the Prometheus operator a short while after
`kubectl create -f manifests/`. If `port-forward` reports "no resources" or the service
isn't found yet, wait ~1 minute and confirm with
`kubectl get pods -n monitoring | grep alertmanager` before retrying.

Now, open a new terminal and let's see what issues Holmes identifies:
```bash
# Run Holmes without a runbook to see current issues
holmes investigate alertmanager --alertmanager-url http://localhost:9093
```

**Note:** When the cluster is freshly created, the only alert that fires immediately is
the built-in `Watchdog` heartbeat. Workload alerts such as `KubePodCrashLooping` and
`KubeDeploymentRolloutStuck` only fire after their `for:` duration elapses (often
10–15 minutes). Give the cluster time, or deploy a broken workload (e.g.
`kubectl create deployment bad-image --image=nginx:nonexistent`) and wait, to see more
alerts to investigate.

So on a fresh cluster Holmes simply confirms the `Watchdog` alert is healthy. The
`bad-image` deployment you created earlier will eventually trip a workload alert once its
`for:` timer elapses, at which point `holmes investigate` will pick it up too.

#### Focus and deepen an investigation

You don't have to investigate every firing alert at once. These flags let you target and
expand an investigation:

```bash
# Investigate only one named alert (regex is supported for the name)
holmes investigate alertmanager --alertmanager-url http://localhost:9093 \
  --alertmanager-alertname Watchdog
```

```bash
# Increase verbosity to watch every tool call Holmes makes (-v, -vv, or -vvv)
holmes investigate alertmanager --alertmanager-url http://localhost:9093 -vv
```

#### Guiding an investigation with your own instructions (runbooks)

A *runbook* is simply a codified set of steps you'd normally take by hand to investigate a
known problem. With the CLI you give Holmes a runbook by writing those steps, in plain
English, directly into your question to `holmes ask`. Give Holmes an explicit, numbered
procedure for the broken workload you deployed earlier:

```bash
holmes ask "Investigate the bad-image deployment by following these steps:
1. Check the pod status and recent events
2. Read the pod logs and any previous logs
3. Determine whether this is an image, resource, or configuration problem
4. Report the root cause and the smallest change that would fix it"
```

Notice that Holmes follows your numbered steps in order — exactly the way it would follow a
runbook. Save prompts like this in a text file and reuse them as your team's standard
procedures for the alerts you see most often.

Tips for writing investigation instructions:
1. Run `holmes investigate alertmanager` (or `holmes ask`) **without** guidance first to see
   what Holmes finds on its own.
2. Note the recurring alerts and failure patterns in your cluster.
3. Write down the exact steps you'd take by hand for each one.
4. Feed those steps to Holmes as a numbered procedure, then refine them based on the results.

### Challenge Tasks

1. **Target a specific alert.** Once a workload alert is firing, scope the investigation to
   just that alert by name (the value accepts a regex):
   ```bash
   holmes investigate alertmanager --alertmanager-url http://localhost:9093 \
     --alertmanager-alertname KubePodCrashLooping
   ```

2. **Customize investigation depth.** Re-run an investigation at different verbosity levels
   and compare how much of the tool-calling you can see:
   ```bash
   holmes investigate alertmanager --alertmanager-url http://localhost:9093 -vv
   holmes investigate alertmanager --alertmanager-url http://localhost:9093 -vvv
   ```

3. **Write a complex, branching procedure.** Ask Holmes to follow a conditional workflow,
   the kind a real runbook encodes:
   ```bash
   holmes ask "Investigate any not-ready pods in the default namespace using this workflow:
   - First check the owning deployment's status and events.
   - If the pods are running but not ready: check the readiness and liveness probes and verify service connectivity.
   - If the pods are not running: check node resources, image pull status, and any PVCs.
   - Finally, summarize all issues found, prioritize the critical ones, and suggest remediation steps."
   ```

### Lab Completion

Congratulations! You have completed the HolmesGPT lab. You have learned:
- How to install and configure HolmesGPT using Helm
- Different LLM backend configurations
- How to create and use custom runbooks for automated investigations

Key takeaways:
1. HolmesGPT can be easily deployed using Helm
2. Multiple LLM backends are supported
3. Custom runbooks help standardize troubleshooting across teams
4. Plain English instructions can guide AI investigations
