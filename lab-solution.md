### Part 1: Introduction and Setup for K8sGPT

#### Introduction

##### **Overview of the Lab Objectives**
- Install and configure K8sGPT with different AI backends
- Master K8sGPT's core troubleshooting capabilities
- Learn advanced filtering and analysis techniques
- Integrate K8sGPT with existing tooling
- Compare manual debugging with K8sGPT-assisted debugging
- Understand security scanning and compliance features

##### **Brief on K8sGPT**
- **K8sGPT**: An AI-powered tool that enhances Kubernetes troubleshooting and management
- **Key Features**: 
  - Automated problem detection and analysis
  - Natural language explanations of issues
  - Suggested remediation steps
  - Security vulnerability scanning
  - Integration with multiple AI backends
  - Custom filtering and rules
  - Real-time monitoring capabilities

##### **Tools and Technologies Required**
- **Kubernetes Cluster**: A running Kubernetes cluster (e.g., AKS, minikube, or kind)
- **kubectl**: Command-line tool for Kubernetes
- **K8sGPT**: The K8sGPT CLI tool
- **OpenAI API Key**: For OpenAI backend integration
- **Helm**: For installing certain components
- **jq**: Command-line JSON processor

#### Environment Setup

##### **Install Required Tools**

###### Install jq
```bash
# For macOS users
brew install jq

# For Linux users (Debian/Ubuntu)
sudo apt-get update && sudo apt-get install -y jq

# For Linux users (RHEL/CentOS)
sudo yum install -y jq

# For Windows users (using Chocolatey)
choco install jq
```

###### Verify jq Installation
```bash
jq --version
```

##### **Install K8sGPT**
```bash
# For macOS users
brew install k8sgpt

# For Linux users
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/download/v0.3.8/k8sgpt_amd64
chmod +x k8sgpt_amd64
sudo mv k8sgpt_amd64 /usr/local/bin/k8sgpt

# For Windows users (PowerShell)
curl.exe -LO https://github.com/k8sgpt-ai/k8sgpt/releases/download/v0.3.8/k8sgpt_windows_amd64.exe
Move-Item .\k8sgpt_windows_amd64.exe k8sgpt.exe
```

##### **Verify Installation**
```bash
k8sgpt version
```

##### **Configure AI Backend**

###### OpenAI Backend
```bash
# Set OpenAI as the backend
k8sgpt auth add --backend openai --password <your-api-key> --model gpt-3.5-turbo

# Verify configuration
k8sgpt auth list
```

### Part 2: Basic K8sGPT Operations

#### Deploy Sample Workloads

Before we start analyzing with K8sGPT, let's deploy some sample workloads with common issues:

```bash
# Deploy a pod with resource constraints
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
        memory: "1000Gi"  # Intentionally requesting too much memory
EOF

# Deploy a pod with an invalid image
kubectl create deployment bad-image --image=nginx:nonexistent

# Deploy a service without matching pods
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: missing-backend
spec:
  selector:
    app: nonexistent
  ports:
  - port: 80
EOF
```

#### Initial Cluster Analysis

Now that we have some issues to find, let's use K8sGPT to analyze them:

##### **Perform Basic Cluster Check**
```bash
# Run initial analysis
k8sgpt analyze

# Save results to file and analyze them
k8sgpt analyze --output json > initial-analysis.json

# View the structured output
cat initial-analysis.json | jq
```

The JSON output allows us to:
- Parse and process the results programmatically
- Integrate with other tools and workflows
- Create custom reports and dashboards
- Track issues over time by comparing analysis results

For example, you could create a simple script to monitor trends:
```bash
#!/bin/bash
# Save hourly analyses
mkdir -p k8sgpt-history
while true; do
    timestamp=$(date +%Y%m%d_%H%M%S)
    k8sgpt analyze --output json > k8sgpt-history/analysis_$timestamp.json
    sleep 3600
done
```

You should see K8sGPT detect multiple issues:
- The pod that can't be scheduled due to resource constraints
- The deployment failing due to an invalid image
- The service with no matching endpoints

##### **Understanding K8sGPT Output**

The default CLI output shows:
- A numbered list of issues
- The namespace/name of the affected resource
- The parent object (if applicable)
- The error message describing the issue

For example:
```
0 default/missing-backend(missing-backend)
- Error: Service has no endpoints, expected label app=nonexistent

1 default/resource-heavy-pod(resource-heavy-pod)
- Error: 0/1 nodes are available: 1 Insufficient memory...

2 default/bad-image-789bb667f5-29h2c(Deployment/bad-image)
- Error: Back-off pulling image "nginx:nonexistent"
```

The JSON output (`--output json`) provides more structured data:
```json
{
  "provider": "openai",          // The AI backend being used
  "errors": null,               // Any errors in the analysis process
  "status": "ProblemDetected", // Overall analysis status
  "problems": 3,               // Total number of problems found
  "results": [                 // Array of detected issues
    {
      "kind": "Service",      // Resource type
      "name": "default/missing-backend",  // Resource name
      "error": [              // Array of errors
        {
          "Text": "Service has no endpoints...",
          "KubernetesDoc": "",
          "Sensitive": []     // Masked sensitive information
        }
      ],
      "details": "",
      "parentObject": "missing-backend"
    }
    // ... additional results
  ]
}
```

Let's analyze this output:
```bash
# Count issues by resource kind
cat initial-analysis.json | jq '.results | group_by(.kind) | map({kind: .[0].kind, count: length})'

# List all error messages
cat initial-analysis.json | jq -r '.results[].error[].Text'

# Show only high-priority issues (pods with errors)
cat initial-analysis.json | jq -r '.results[] | select(.kind=="Pod")'
```

#### Filtering and Customization

Now let's learn how to focus on specific issues:

##### **Namespace Filtering**
```bash
# Analyze only our test deployments in default namespace
k8sgpt analyze --namespace default
```

##### **Resource Filtering**
```bash
# Look only at pod issues
k8sgpt analyze --filter Pod

# Check only service issues
k8sgpt analyze --filter Service
```

### Part 3: Practical Troubleshooting Scenarios

#### Scenario 1: Image Pull Failures

##### **Manual Debugging Approach**
Let's walk through how we would traditionally debug this issue:

```bash
# Check pod status
kubectl get pods
```
This shows us the high-level status of our pods. You should see something like:
```
NAME                         READY   STATUS             RESTARTS   AGE
bad-image-789bb667f5-29h2c   0/1     ImagePullBackOff   0          2m
```
The `ImagePullBackOff` status indicates that Kubernetes is having trouble pulling the container image and is backing off (waiting longer between retries) to avoid overwhelming the system.

```bash
# Describe pod for detailed information
POD_NAME=$(kubectl get pods -l app=bad-image -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD_NAME
```
The output shows several important pieces of information:
1. Pod Details:
   - Name and namespace
   - Node it's scheduled on
   - Labels and annotations
   - IP address

2. Container Status:
   ```
   State:          Waiting
      Reason:       ImagePullBackOff
   Ready:          False
   ```
   This tells us the container is waiting because it can't pull the image `nginx:nonexistent`

3. Events:
   ```
   Events:
     Type    Reason   Age                     From     Message
     ----    ------   ----                    ----     -------
     Normal  BackOff  2m38s (x484 over 112m)  kubelet  Back-off pulling image "nginx:nonexistent"
   ```
   The events show that the kubelet has been repeatedly trying and failing to pull the image.

```bash
# Check cluster-wide events
kubectl get events --sort-by='.lastTimestamp'
```
This shows recent events across the cluster, sorted by time. In this case:
```
LAST SEEN   TYPE      REASON             OBJECT                           MESSAGE
3m4s        Normal    BackOff            pod/bad-image-789bb667f5-29h2c   Back-off pulling image "nginx:nonexistent"
```

From this manual investigation, we can determine:
1. The pod can't start because it's trying to pull a nonexistent image tag
2. The kubelet is in a backoff state to avoid overwhelming the container registry
3. To fix this, we would need to specify a valid nginx image tag

In the next section, we'll see how K8sGPT can automate this analysis and provide remediation steps.

##### **K8sGPT Analysis**
```bash
# Quick analysis
k8sgpt analyze --filter Pod

# Detailed analysis
k8sgpt analyze --filter Pod --explain
```

#### Scenario 2: Resource Constraints

##### **Create Resource-Constrained Pod**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-test-pod
spec:
  containers:
  - name: memory-demo
    image: nginx
    resources:
      requests:
        memory: "1000Gi"
        cpu: "100"
EOF
```

##### **Manual Investigation**
```bash
# Check pod status
kubectl get pod resource-test-pod

# Check node resources
kubectl describe nodes

# View scheduling events
kubectl describe pod resource-test-pod
```

##### **K8sGPT Analysis**
```bash
# Analyze scheduling issues
k8sgpt analyze --filter Pod --explain

# Check node capacity issues
k8sgpt analyze --filter Node

# Generate comprehensive report
k8sgpt analyze --output markdown > resource-analysis.md
```

##### **K8sGPT Analysis with Explanations**
```bash
k8sgpt analyze --explain
```

This will show detailed explanations for each issue. For example:

```
AI Provider: openai

0 default/bad-image-789bb667f5-29h2c(Deployment/bad-image)
- Error: Back-off pulling image "nginx:nonexistent"
Error: The Kubernetes pod is unable to pull the image "nginx:nonexistent" due to a back-off error.

Solution:
1. Check if the image "nginx:nonexistent" exists in the specified repository.
2. Correct the image name or pull a different image if necessary.
3. Update the pod configuration with the correct image name.
4. Restart the pod to try pulling the correct image again.

1 default/resource-heavy-pod(resource-heavy-pod)
- Error: 0/1 nodes are available: 1 Insufficient memory. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..
Error: 0/1 nodes are available due to insufficient memory. No preemption victims found for incoming pod.

Solution:
1. Check memory usage on existing nodes.
2. Increase memory resources on nodes or add more nodes to the cluster.
3. Ensure proper resource limits are set for pods.
```

The `--explain` flag provides:
- A clear explanation of what the error means in plain language
- Step-by-step solutions to resolve the issue
- Actionable recommendations for fixing the problem

This is much more user-friendly than parsing through logs and events manually, especially for those new to Kubernetes troubleshooting.

### Part 4: Security Scanning with Trivy Integration

K8sGPT can integrate with Trivy to provide vulnerability and configuration scanning.

#### Enable Trivy Integration
```bash
# Check available integrations
k8sgpt integrations list

# Activate Trivy
k8sgpt integration activate trivy
```

#### Security Analysis

##### **Vulnerability Scanning**
```bash
# Check for vulnerabilities in your cluster
k8sgpt analyze --filter VulnerabilityReport
```
This will show CVEs found in your container images, including severity levels and links to detailed information.

##### **Configuration Auditing**
```bash
# Check for security misconfigurations
k8sgpt analyze --filter ConfigAuditReport
```
This will identify security issues in your Kubernetes configurations, such as:
- Containers running as root
- Missing security contexts
- Privileged containers
- Unsafe volume mounts
- Missing seccomp profiles

Each issue includes a severity rating (LOW/MEDIUM/HIGH) and specific remediation advice.

### Part 5: Prometheus Integration for Metrics Analysis

#### Install Prometheus
First, let's install Prometheus using kube-prometheus:

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

#### Enable Prometheus Integration
Once Prometheus is running, integrate it with K8sGPT:

```bash
# Activate Prometheus integration
k8sgpt integration activate prometheus --namespace monitoring
```

This will allow K8sGPT to analyze metrics from your cluster components.

### Lab Completion

Congratulations! You have completed the K8sGPT lab. You have learned:
- How to install and configure K8sGPT
- The difference between manual and K8sGPT-assisted debugging
- Advanced features of K8sGPT
- Real-world troubleshooting scenarios

Key takeaways:
1. K8sGPT significantly speeds up Kubernetes troubleshooting
2. Natural language explanations make issues more accessible
3. Automated analysis can catch issues that might be missed manually

Feel free to explore more features or proceed to additional labs to deepen your understanding of Kubernetes troubleshooting.