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

Note: If you have resources deployed from a previous lab, run `minikube delete` to start with a fresh cluster.

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

##### **Additional Test Scenarios**

###### **Scenario 1: Fixing Resource Configuration Issues**

First, deploy a pod with resource configuration issues:
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
        memory: "2Gi"
      limits:
        memory: "4Gi"
EOF
```

Ask HolmesGPT to analyze the pod and suggest appropriate resource limits:
```bash
holmes ask "analyze memory-heavy-pod and suggest appropriate memory limits based on nginx best practices. ONLY PROVIDE USER-CONFIGURABLE FIELDS"
```

Apply the fixed YAML suggested by HolmesGPT:
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

##### **Using HolmesGPT for Complex Analysis**

###### **Security Analysis**
```bash
# Check for security vulnerabilities
holmes ask "are there any pods running with privileged security context?"

# Analyze network policies
holmes ask "what network policies are in place and are they properly configured?"

# Check RBAC configuration
holmes ask "analyze the RBAC permissions in my cluster"
```

###### **Performance Investigation**
```bash
# Resource utilization analysis
holmes ask "which pods are consuming the most resources?"

# Node analysis
holmes ask "are there any nodes under heavy load?"

# Scaling analysis
holmes ask "analyze the HPA configuration and scaling patterns"
```

###### **Configuration Audit**
```bash
# Best practices check
holmes ask "are there any pods not following Kubernetes best practices?"

# Resource quotas analysis
holmes ask "analyze resource quotas and limits across all namespaces"

# Storage configuration
holmes ask "check for any storage-related issues or misconfigurations"
```

##### **Troubleshooting Common Issues**

###### **Investigating CrashLoopBackOff**
```bash
# Get detailed analysis of crashing pods
holmes ask "why is pod $POD_NAME in CrashLoopBackOff?"

# Check container logs
holmes ask "show me the logs for the crashing container in pod $POD_NAME"
```

###### **Network Connectivity Issues**
```bash
# Service connectivity
holmes ask "can service A reach service B?"

# DNS resolution
holmes ask "are there any DNS resolution issues in the cluster?"
```

###### **Resource Constraints**
```bash
# Node pressure analysis
holmes ask "are any nodes experiencing resource pressure?"

# Pod eviction analysis
holmes ask "why are pods being evicted?"
```

##### **Integration with Other Tools**

###### **Using with kubectl**
```bash
# Combine kubectl output with Holmes analysis
kubectl get pods -A | holmes ask "analyze this pod list for potential issues"

# Analyze events
kubectl get events --all-namespaces | holmes ask "what significant events should I be concerned about?"
```

###### **Using with Logs**
```bash
# Analyze application logs
kubectl logs $POD_NAME | holmes ask "what errors are present in these logs?"

# Analyze system logs
kubectl logs -n kube-system $SYSTEM_POD | holmes ask "analyze these system logs for issues"
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

First, set up port forwarding for AlertManager:
```bash
kubectl port-forward -n monitoring svc/alertmanager-main 9093:9093
```

Now, open a new terminal and let's see what issues Holmes identifies:
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

Then run Holmes with your runbook:
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

### Challenge Tasks

1. Add more alert types to your runbooks
```bash
   # Example of adding more alert types
   cat <<EOF >> custom_runbooks.yaml
runbooks:
   - match:
       issue_name: "KubeSchedulerDown"
     instructions: >
       Check if the cluster is a managed cluster like EKS by fetching nodes and looking at their labels
       If so, tell the user this is likely a known false positive in the kube-prometheus-stack alert
       If not, check scheduler status and logs
EOF
```

2. Customize investigation depth
   ```bash
   # Test with different verbosity levels
   holmes investigate alertmanager --alertmanager-url http://localhost:9093 -r custom_runbooks.yaml -vv
   holmes investigate alertmanager --alertmanager-url http://localhost:9093 -r custom_runbooks.yaml -vvv

   # Test focused investigation
   holmes investigate alertmanager --alertmanager-url http://localhost:9093 -r custom_runbooks.yaml --alertmanager-alertname KubePodCrashLooping
   ```

3. Create a runbook for a complex scenario
```bash
   cat <<EOF > custom_runbooks.yaml
runbooks:
  - match:
      issue_name: "KubePodNotReady"
    instructions: >
      # First check deployment status
      Check deployment status and events
      Check pod status for this deployment
      
      # If pods are running but not ready
      If pods are running:
        Check readiness probe status
        Check liveness probe status
        Verify service connectivity
      
      # If pods are not running
      If pods are not running:
        Check node resources
        Check image pull status
        Verify PVC status if applicable
        
      # Report findings
      Summarize all discovered issues
      Prioritize critical problems
      Suggest remediation steps
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
