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

K8sGPT can be used in two ways:
1. **CLI Tool**: For interactive troubleshooting and on-demand analysis
2. **Operator**: For continuous monitoring and automated analysis


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
# For Linux users (Debian/Ubuntu)
sudo apt-get update && sudo apt-get install -y jq
```

###### Verify jq Installation
```bash
jq --version
```

##### **Install Helm**

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

#### Install K8sGPT (Both Methods)

##### Install K8sGPT CLI

Download the latest release:
```bash
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/download/v0.3.48/k8sgpt_amd64.deb
```

Install the package:
```bash
sudo dpkg -i k8sgpt_amd64.deb
```

Verify installation:
```bash
k8sgpt version
```

Configure OpenAI backend for CLI:
```bash
k8sgpt auth add --backend openai --password <your-api-key> --model gpt-4o-mini
```

Verify configuration:
```bash
k8sgpt auth list
```

###### Install K8sGPT Operator

Add the K8sGPT Helm repository:
```bash
helm repo add k8sgpt https://charts.k8sgpt.ai/
```

Update Helm repositories:
```bash
helm repo update
```

Install the operator:
```bash
helm install release k8sgpt/k8sgpt-operator -n k8sgpt-operator-system --create-namespace
```

###### Configure OpenAI Backend

Set your OpenAI API key:
```bash
export OPENAI_TOKEN=YOUR_API_KEY
```

Create the Kubernetes secret:
```bash
kubectl create secret generic k8sgpt-sample-secret \
    --from-literal=openai-api-key=$OPENAI_TOKEN \
    -n k8sgpt-operator-system
```

Deploy the K8sGPT resource:
```bash
kubectl apply -f - << EOF
apiVersion: core.k8sgpt.ai/v1alpha1
kind: K8sGPT
metadata:
  name: k8sgpt-sample
  namespace: k8sgpt-operator-system
spec:
  ai:
    enabled: true
    model: gpt-4o-mini
    backend: openai
    secret:
      name: k8sgpt-sample-secret
      key: openai-api-key
  noCache: false
  version: v0.3.41
EOF
```

### Part 2: Basic K8sGPT Operations

#### Deploy Sample Workloads

Deploy a pod with resource constraints:
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

Deploy a pod with an invalid image:
```bash
kubectl create deployment bad-image --image=nginx:nonexistent
```

Deploy a service without matching pods:
```bash
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

Run initial analysis:
```bash
k8sgpt analyze
```

Save results to file:
```bash
k8sgpt analyze --output json > initial-analysis.json
```

View the structured output:
```bash
cat initial-analysis.json | jq
``` 

The JSON output allows us to:
- Parse and process the results programmatically
- Integrate with other tools and workflows
- Create custom reports and dashboards
- Track issues over time by comparing analysis results

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
cat initial-analysis.json | jq '.results | group_by(.kind) | map({kind: .[0].kind, count: length})'
```

List all error messages:
```bash
cat initial-analysis.json | jq -r '.results[].error[].Text'
```

Show only pod-related issues:
```bash
cat initial-analysis.json | jq -r '.results[] | select(.kind=="Pod")'
```

#### Filtering and Customization

Analyze default namespace:
```bash
k8sgpt analyze --namespace default
```

Look only at pod issues:
```bash
k8sgpt analyze --filter Pod
```

Check only service issues:
```bash
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

Using the CLI (on-demand analysis):
```bash
# Analyze scheduling issues
k8sgpt analyze --filter Pod --explain
```

Using the Operator (continuous monitoring):

View all analysis results:
```bash
kubectl get results -n k8sgpt-operator-system
```

View detailed results in JSON format:
```bash
kubectl get results -o json -n k8sgpt-operator-system | jq
```

View specific result details:
```bash
kubectl describe result defaultmissingbackend -n k8sgpt-operator-system
```

Example outputs:

CLI Analysis (`k8sgpt analyze --explain`):
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
- Error: 0/1 nodes are available: 1 Insufficient memory...
Solution:
1. Check memory usage on existing nodes.
2. Increase memory resources on nodes or add more nodes to the cluster.
3. Ensure proper resource limits are set for pods.
```

Operator Analysis (`kubectl get results -o yaml -n k8sgpt-operator-system`):
```yaml
apiVersion: core.k8sgpt.ai/v1alpha1
kind: Result
metadata:
  name: result-sample
  namespace: k8sgpt-operator-system
spec:
  error: Back-off pulling image "nginx:nonexistent"
  kind: Pod
  name: bad-image-789bb667f5-29h2c
  namespace: default
  parent: Deployment/bad-image
  explanation: The Kubernetes pod is unable to pull the image...
  remediation:
    - Check if the image exists in the repository
    - Correct the image name or tag
    - Update the deployment configuration
```

Both tools provide similar insights, but:
- CLI: Immediate, interactive analysis
- Operator: Continuous monitoring and historical tracking of issues

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

Check pod status:
```bash
kubectl get pod resource-test-pod
```

Check node resources:
```bash
kubectl describe nodes
```

View scheduling events:
```bash
kubectl describe pod resource-test-pod
```

##### **K8sGPT Analysis**

Analyze scheduling issues:
```bash
k8sgpt analyze --filter Pod --explain
```

Check node capacity issues:
```bash
k8sgpt analyze --filter Node
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

Check available integrations:
```bash
k8sgpt integrations list
```

Activate Trivy:
```bash
k8sgpt integration activate trivy
```

#### Security Analysis

After activating Trivy integration:

Check Trivy operator deployment status:
```bash
helm list
```

Example output:
```
NAME                 	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART                	APP VERSION
trivy-operator-k8sgpt	default  	1       	2025-01-20 20:36:02.236249031 +0000 UTC	deployed	trivy-operator-0.25.0	0.23.0
```

Once the operator is running, wait a few minutes for initial scans to complete.
Then check for vulnerabilities and misconfigurations:

Check for vulnerabilities:
```bash
k8sgpt analyze --filter VulnerabilityReport
```

Check for security misconfigurations:
```bash
k8sgpt analyze --filter ConfigAuditReport
```

Note: The initial scan may take a few minutes to complete after the Trivy operator is deployed.

<!-- ### Part 5: Prometheus Integration for Metrics Analysis

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

This will allow K8sGPT to analyze metrics from your cluster components. -->

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