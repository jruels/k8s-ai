### Part 1: Introduction and Setup for Botkube

#### Prerequisites

Start minikube with recommended memory:
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

##### **Overview of Lab Objectives**
- Install and configure Botkube with Slack integration
- Set up Slack workspace for Botkube
- Configure Botkube in Kubernetes cluster
- Test Botkube-Slack communication

##### **Brief on Botkube**
- **Botkube**: A Kubernetes monitoring and management tool that integrates with Slack
- **Key Features**:
  - Slack integration for cluster monitoring
  - Interactive messaging
  - Multi-cluster support
  - Command execution through Slack

#### Environment Setup

##### **Install Botkube CLI**

Download the binary:
```bash
curl -Lo botkube https://github.com/kubeshop/botkube/releases/download/v1.14.0/botkube-linux-amd64
```

Make it executable:
```bash
chmod +x botkube
```

Move to system path:
```bash
sudo mv botkube /usr/local/bin/botkube
```

Verify installation:
```bash
botkube version
```

### Part 2: Configure Slack Tokens

1. Get the Bot Token:
   ```bash
   export SLACK_API_BOT_TOKEN="xoxb-your-token"
   ```

2. Get the App-Level Token:
   ```bash
   export SLACK_API_APP_TOKEN="xapp-your-token"
   ```

3. Add Botkube to your Slack channel by inviting `@Botkube`

### Part 4: Deploy Botkube to Cluster

Install Botkube with Slack integration:
```bash
export CLUSTER_NAME="<your-name>-cluster"
export ALLOW_KUBECTL=true
export SLACK_CHANNEL_NAME="<your-channel-name>"

botkube install --version v1.14.0 \
--set communications.default-group.socketSlack.enabled=true \
--set communications.default-group.socketSlack.channels.default.name=${SLACK_CHANNEL_NAME} \
--set communications.default-group.socketSlack.appToken=${SLACK_API_APP_TOKEN} \
--set communications.default-group.socketSlack.botToken=${SLACK_API_BOT_TOKEN} \
--set settings.clusterName=${CLUSTER_NAME} \
--set 'executors.k8s-default-tools.botkube/kubectl.enabled'=${ALLOW_KUBECTL}
```

### Part 5: Test Botkube

In your Slack channel, explore Botkube's capabilities:

1. Get started with help:
```
@Botkube help
```

2. Explore available plugins:
```
@Botkube list source plugins
@Botkube list executor plugins
@Botkube list filter plugins
```

3. Check notification settings:
```
@Botkube notifications get status
@Botkube notifications enable
@Botkube notifications disable
```

4. Try kubectl commands using the interactive composer:
```
@Botkube kubectl
```
This will open an interactive menu where you can:
- Select command categories (api-resources, api-versions, cluster-info, describe, get, logs, top)
- Choose resource types (deployments, pods, namespaces, daemonsets, statefulsets, storageclasses, nodes, configmaps)
- Get example commands like 'kubectl describe deployments -n default'
- Add filters to refine the output

> **Note**: The interactive composer makes it easy to build and execute kubectl commands without remembering the exact syntax.

### Part 6: Adding Helm Plugin

Upgrade your Botkube installation to include Helm support:
```bash
botkube install --version v1.14.0 \
--set communications.default-group.socketSlack.enabled=true \
--set communications.default-group.socketSlack.channels.default.name=${SLACK_CHANNEL_NAME} \
--set communications.default-group.socketSlack.appToken=${SLACK_API_APP_TOKEN} \
--set communications.default-group.socketSlack.botToken=${SLACK_API_BOT_TOKEN} \
--set settings.clusterName=${CLUSTER_NAME} \
--set 'executors.k8s-default-tools.botkube/kubectl.enabled'=${ALLOW_KUBECTL} \
--set 'executors.helm.botkubeExtra/helm.enabled'=true \
--set 'executors.helm.botkubeExtra/helm.config.defaultNamespace'=default \
--set 'plugins.repositories.botkubeExtra.url'=https://github.com/kubeshop/botkube-plugins/releases/download/v1.14.0/plugins-index.yaml
```

Test Helm integration with these read-only commands:
```
@Botkube helm help
@Botkube helm list
@Botkube helm status <release-name>
@Botkube helm version
```

For filtered listing, use:
```
@Botkube helm list -f 'ara[a-z]+'
```

Note: The following read-write commands are also available if RBAC is properly configured:
```
@Botkube helm rollback <release-name> <revision>
@Botkube helm test <release-name>
@Botkube helm uninstall <release-name>
@Botkube helm upgrade <release-name> <chart>
@Botkube helm install <release-name> <chart>
```

> **Note**: The `--wait` flag is not supported by the Botkube Helm plugin. Also, when using flags, ensure there's a space between the flag and its value (use `-o yaml` instead of `-oyaml`).

### Part 7: Cleanup

Remove Botkube from Slack:
1. Go to [Slack Apps page](https://api.slack.com/apps)
2. Click on "Botkube"
3. Scroll to bottom and click "Delete App"

Remove Botkube from cluster:
```bash
botkube uninstall
```

### Lab Completion

Congratulations! You have completed the Botkube lab. You have learned:
- How to set up Botkube with Slack integration
- How to configure Slack app for Botkube
- How to deploy Botkube to Kubernetes
- How to test Botkube-Slack communication

Key takeaways:
1. Botkube provides Slack integration for Kubernetes monitoring
2. Socket mode enables interactive messaging
3. Multiple clusters require separate Botkube apps in Slack                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            