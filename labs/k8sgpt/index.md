### K8sGPT Lab: AI-Powered Kubernetes Troubleshooting

#### Prerequisites

Start minikube with 6GB memory.

#### Introduction

##### **Overview of Lab Objectives**
- Install and configure K8sGPT
- Compare manual vs AI-assisted debugging
- Explore different analysis modes
- Integrate with security scanning tools

For detailed documentation, see: https://docs.k8sgpt.ai/

##### **Brief on K8sGPT**
- **K8sGPT**: An AI-powered tool for Kubernetes cluster analysis
- **Key Features**:
  - Real-time cluster analysis
  - Natural language explanations
  - Multiple deployment options (CLI and Operator)
  - Security scanning integration
  - Continuous monitoring capabilities

#### Lab Tasks

### Part 1: Installing K8sGPT

1. Install K8sGPT CLI
   - Download and install K8sGPT
   - Configure your AI provider for CLI use
   - Configure your AI provider for Operator use
   - Verify the installation
   - Test filtering output with jq

2. Deploy K8sGPT Operator
   - Install the operator using Helm
   - Configure necessary permissions
   - Verify operator deployment

### Part 2: Basic Troubleshooting

1. Create test scenarios
   - Deploy a pod with an invalid image
   - Create a pod with resource constraints (2000Gi memory)
   - What issues do you expect K8sGPT to find?

2. Compare troubleshooting approaches
   - Investigate issues manually
   - Use K8sGPT to analyze the same issues
   - Document the differences in approach and results

3. Compare CLI vs Operator approaches
   - Test basic analysis with CLI
   - Test same scenario with Operator
   - Use analyze vs analyze --explain
   - Document key differences

4. Test different analysis modes
   - Try filtering by resource types
   - Use the explain flag
   - Explore continuous monitoring

### Part 3: Security Integration

1. Set up security scanning
   - Enable Trivy integration
   - Configure vulnerability scanning
   - Set up ConfigAudit reports
   - Test security analysis features

2. Analyze security reports
   - Check for vulnerabilities
   - Review configuration issues
   - Understand remediation suggestions

### Success Criteria

1. You can install and configure K8sGPT
2. You understand the difference between manual and AI-assisted debugging
3. You can use both CLI and Operator effectively
4. You can integrate security scanning
5. You can interpret and act on K8sGPT's suggestions