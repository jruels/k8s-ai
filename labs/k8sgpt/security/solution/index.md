### Part 1: Introduction and Setup for K8sGPT

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