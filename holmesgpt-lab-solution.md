### Part 1: Installing HolmesGPT with Helm

#### Prerequisites

Start minikube with 6GB memory:
```bash
minikube start --memory=6144
```

If minikube is already running, you can update its resources:
```bash
minikube stop
minikube delete
minikube start --memory=6144
```

#### Introduction

##### **Overview of the Lab Objectives**
- Install Helm and HolmesGPT
- Configure different LLM backends
- Test different investigation scenarios
- Understand security and configuration options

#### Installing Helm

First, we need to install Helm. For Debian/Ubuntu systems:

Download the installation script:
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```

Make the script executable:
```bash
chmod 700 get_helm.sh
```

Run the installation script:
```bash
./get_helm.sh
```

Verify the installation:
```bash
helm version
```

##### **Brief on HolmesGPT**
- **HolmesGPT**: An AI-powered tool for investigating and resolving Kubernetes alerts
- **Key Features**:
  - Automatic data collection
  - Read-only access with RBAC permissions
  - Runbook automation
  - Multiple LLM backend support
  - Integration with existing tools (PagerDuty, etc.)

#### Environment Setup

##### **Create Configuration File**

Create a values file for Helm configuration:
```bash
cat <<EOF > holmes-values.yaml
additionalEnvVars:
- name: MODEL
  value: gpt-4o
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
  --from-literal=openai-key='your-openai-key-here'
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
  name: resource-heavy-pod
spec:
  containers:
  - name: memory-demo
    image: nginx
    resources:
      requests:
        memory: "1000Gi"
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
pipx install "https://github.com/robusta-dev/holmesgpt/archive/refs/tags/0.7.2.zip"
```

Update your PATH:
```bash
pipx ensurepath
source ~/.bashrc  # or restart your terminal
```

##### **Configure HolmesGPT**

You can either:

1. Use the config file:
```bash
mkdir -p ~/.holmes
cat <<EOF > ~/.holmes/config.yaml
model: "gpt-4"
api_key: "your-api-key-here"
EOF
```

2. Or use the command line flag:
```bash
holmes ask --api-key="your-openai-key" "what pods are unhealthy in my cluster?"
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
holmes ask "why is my resource-heavy-pod pending?"
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

# Wait for CRDs to be ready
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done

# Create the monitoring stack
kubectl create -f manifests/
```

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

#### Create Custom Runbooks

First, let's see what issues Holmes identifies:
```bash
# Run Holmes without a runbook to see current issues
holmes investigate alertmanager --alertmanager-url http://localhost:9093
```

You'll see several issues like "KubePodCrashLooping", "KubeDeploymentRolloutStuck", etc. Let's create a runbook for a couple of these:

```bash
# Create an initial runbook with a few alerts
cat <<EOF > custom_runbooks.yaml
runbooks:
  - match:
      issue_name: "KubePodCrashLooping"
    instructions: >
      Check pod status and events
      Check pod logs for error messages
      Check previous pod logs if available
      Report any resource constraints or configuration issues

  - match:
      issue_name: "KubeDeploymentRolloutStuck"
    instructions: >
      Check deployment rollout status
      Check pod events for scheduling issues
      Verify image pull status
      Check for resource constraints
EOF
```

Now run Holmes with your runbook:
```bash
holmes investigate alertmanager --alertmanager-url http://localhost:9093 -r custom_runbooks.yaml
```

Notice how Holmes now follows your instructions for these specific alerts. For other alerts, it will use its default behavior.

You can gradually add more alerts to your runbook as you identify common issues. For example, if you see "KubePodNotReady" alerts, you might add:

```bash
cat <<EOF >> custom_runbooks.yaml
  - match:
      issue_name: "KubePodNotReady"
    instructions: >
      Check pod status and events
      Verify resource requests and limits
      Check node capacity and allocatable resources
      Report any scheduling or resource issues
EOF
```

Tips for writing runbooks:
1. Use `holmes investigate alertmanager` without a runbook first to see what issues exist
2. Note the `issue_name` values from the alerts
3. Create runbook entries matching those names
4. Add investigation steps that you'd normally take for each issue
5. Test and refine your instructions based on results

A complete runbook might look like this:
```bash
cat <<EOF > custom_runbooks.yaml
runbooks:
  - match:
      issue_name: "KubePodCrashLooping"
    instructions: >
      Check pod status and events
      Check pod logs for error messages
      Check previous pod logs if available
      Report any resource constraints or configuration issues

  - match:
      issue_name: "KubeDeploymentRolloutStuck"
    instructions: >
      Check deployment rollout status
      Check pod events for scheduling issues
      Verify image pull status
      Check for resource constraints

  - match:
      issue_name: "KubePodNotReady"
    instructions: >
      Check pod status and events
      Verify resource requests and limits
      Check node capacity and allocatable resources
      Report any scheduling or resource issues
EOF
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
