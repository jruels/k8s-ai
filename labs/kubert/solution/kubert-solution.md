### Part 1: Introduction and Setup for Kubert Assistant Lite

#### Prerequisites

If minikube is already running, you can stop and delete it:
```bash
minikube stop
minikube delete
```
#### Introduction

##### **Overview of Lab Objectives**
- Install and configure Kubert Assistant Lite
- Deploy a local Kubernetes environment using kind
- Test the Kubectl Agent functionality
- Understand Helm-based deployment configuration

##### **Brief on Kubert Assistant**
- **Kubert Assistant**: A DevOps productivity tool for Kubernetes management
- **Key Features**:
  - Local kind cluster deployment
  - Kubectl Agent for command execution
  - Helm-based deployment
  - AI-powered operations (with OpenAI/Anthropic)

#### Environment Setup

##### **Clone Repository**
```bash
git clone https://github.com/TranslucentComputing/kubert-assistant-lite.git
```

```bash
cd kubert-assistant-lite
```

For testing, initialize BATS submodules:
```bash
git submodule update --init --recursive
```

##### **Install kind**
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
```

```bash
chmod +x ./kind
```

```bash
sudo mv ./kind /usr/local/bin/kind
```

##### **Verify kind Installation**
```bash
kind version
```

##### **Verify Prerequisites**
First, verify prerequisites:
```bash
make check-deps-deploy
```

This checks for:
- Docker
- kind
- Helm
- kubectl
- jq
- Make

### Part 2: Deployment

#### Deploy Cluster and Application

Deploy the kind cluster and Kubert Assistant:
```bash
make deploy
```

This executes `deploy.sh` which:
1. Creates a kind cluster
2. Deploys Kubert Assistant components
3. Updates host file with required domain entries
4. Configures AI provider API keys

> **Note**: During deployment, you will be prompted for several actions:
>
> 1. **Host File Updates** - The following entries will be added to `/etc/hosts`:
>    - kubert-assistant.lan
>    - kubert-agent.lan
>    - kubert-plugin.lan
>    - kubert-plugin-gateway.lan
>    - kubert-ollama.lan
>
> 2. **AI Provider Configuration** - You will be prompted to configure API keys for:
>    - OpenAI (required)
>    - Anthropic (optional)
>    - Groq (optional)
>    - Google AI (optional)
>    - Perplexity AI (optional)

The deployment process will automatically handle DNS configuration and CoreDNS setup within the cluster.

#### Helm Values Configuration

The repository includes pre-configured Helm values files in `manifests/kubert-assistant/`:

1. `gateway-values.yaml` - Defines the gateway service configuration
2. `agent-repo-values.yaml` - Contains the Kubectl agent settings
3. `command-runner-values.yaml` - Manages AI provider configuration

> **Important**: These files are part of the repository and should not be modified manually. The deployment process will automatically use these configurations and apply your AI provider settings based on the prompts.

### Part 3: Accessing the Web UI

After deployment, to access the Kubert Assistant web interface from your local machine, you'll need to:
1. Configure your local DNS entries
2. Set up SSH port forwarding

#### Step 1: Add Local Host Entries
On macOS, edit the hosts file:
```bash
# Open hosts file in nano editor
sudo nano /etc/hosts

# Or with vim if you prefer
sudo vim /etc/hosts
```

Add these entries:
```
127.0.0.1 kubert-assistant.lan
127.0.0.1 kubert-agent.lan
127.0.0.1 kubert-plugin.lan
127.0.0.1 kubert-plugin-gateway.lan
```

Save the file:
- In nano: Press `Ctrl + O` to write the file, then `Ctrl + X` to exit
- In vim: Press `ESC`, type `:wq`, and press `Enter`

#### Step 2: SSH Port Forward
From your local machine, create an SSH tunnel to the VM:
```bash
sudo ssh -L 80:localhost:80 ubuntu@<VM-IP-ADDRESS> -i lab.pem
```

Then access the services in your local browser at:
- Main UI: http://kubert-assistant.lan

> **Note**: Keep the SSH tunnel terminal window open while accessing the services. The connection will close if you close the terminal.

### Part 4: Using Kubert Assistant

Once you have access to the web UI, you can interact with your Kubernetes cluster through natural language:

1. In the web UI, select "kubert - kubectl" from the assistant options
2. Try these commands in sequence to create and inspect resources:
   ```
   create namespace test-kubert
   ```
   This will create a new namespace for our tests.

   ```
   show namespaces
   ```
   Verify that our new namespace was created.

   ```
   create a redis deployment with the redis image in the test-kubert namespace
   ```
   Creates a deployment running Redis in the test-kubert namespace.

   ```
   describe the redis deployment
   ```
   Shows details about the deployment, including the image being used.

The assistant translates these natural language requests into kubectl commands and executes them on your behalf. You can continue to interact with the deployment using natural language commands.

### Part 5: Advanced Kubert Assistant Exercises

#### Exercise 1: Deployment Management
Try these commands to explore deployment management:
```
scale the redis deployment to 3 replicas
get pods to verify the scaling
show the deployment history
rollout status of redis deployment
```

#### Exercise 2: Pod Investigation
Investigate pod details and logs:
```
show logs from redis pods
describe the redis pods
get redis pod metrics
show redis pod resource usage
```

#### Exercise 3: Service Creation and Networking
Set up networking for the deployment:
```
expose the redis deployment on port 6379
describe the redis service
get endpoints for the redis service
```

#### Exercise 4: ConfigMaps and Redis Configuration
Create and manage Redis configuration:
```
create a configmap named redis-config with data "maxmemory=2mb"
show all redis configmaps in the test-kubert namespace
describe the redis-config configmap
```

#### Exercise 5: Resource Quotas and Limits
Manage resource constraints:
```
create resource quota for test-kubert namespace
set memory limit to 512Mi for redis deployment
set cpu limit to 500m for redis deployment
show resource quotas
```

#### Exercise 6: Labels and Selectors
Work with labels and selectors:
```
label redis pods with environment=testing
show pods with the testing label
remove the environment label from redis pods
list all labels in the namespace
```

> **Note**: The beauty of Kubert Assistant is that you can use natural language. If a command doesn't work exactly as written, try rephrasing it or asking the assistant for help.

### Part 6: Cleanup

Remove Kubert Assistant:
```bash
make cleanup-kubert-assistant
```

Delete entire cluster:
```bash
make cleanup
```

### Lab Completion

Congratulations! You have completed the Kubert Assistant Lite lab. You have learned:
- How to deploy a local Kubernetes environment with kind
- How to configure and deploy Kubert Assistant
- How to use the Kubectl Agent
- How to manage deployments with Helm

Key takeaways:
1. Kubert Assistant simplifies Kubernetes management
2. Helm-based deployment enables easy configuration
3. Local testing environment with kind is valuable for development
4. AI integration enhances DevOps workflows

Feel free to explore more features or proceed to additional labs.
