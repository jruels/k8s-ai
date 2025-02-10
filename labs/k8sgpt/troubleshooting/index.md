### K8sGPT Lab: AI-Powered Kubernetes Troubleshooting

#### Prerequisites

Start minikube with 6GB memory.

Note: If you have resources deployed from a previous lab, run `minikube delete` to start with a fresh cluster.

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
   - Create a service without matching pods
   - What issues do you expect K8sGPT to find?

2. Compare troubleshooting approaches
   - Investigate issues manually using kubectl commands
   - Use K8sGPT to analyze the same issues
   - Document the differences in approach and results
   - Try both basic analysis and the --explain flag
   - Save results to JSON for further processing

3. Compare CLI vs Operator approaches
   - Test basic analysis with CLI
   - Test same scenario with Operator
   - Use analyze vs analyze --explain
   - View results using kubectl get results
   - Document key differences

4. Test different analysis modes
   - Try filtering by resource types (Pod, Service, etc.)
   - Use the explain flag for detailed explanations
   - Save and process results using jq
   - Explore continuous monitoring
   - Test namespace-specific analysis

5. Implement fixes for each issue
   - Fix the invalid image deployment using kubectl set image
   - Address the resource constraints by recreating the pod
   - Fix the service by creating matching backend pods
   - Verify fixes using both kubectl and k8sgpt analyze

### Part 3: Advanced Troubleshooting Scenarios

1. Resource Constraints Scenario
   - Create a pod with excessive CPU and memory requests
   - Investigate scheduling issues manually
   - Use K8sGPT to analyze node capacity
   - Compare manual vs AI-assisted debugging approaches
   - Implement and verify fixes

2. Network Policy Issues
   - Create a restrictive default-deny network policy
   - Deploy test pods and services
   - Analyze policy impact using K8sGPT
   - Implement more specific network policies
   - Test and verify connectivity

3. Storage Problems
   - Create PVC with nonexistent storage class
   - Deploy pod using the problematic PVC
   - Analyze storage issues with K8sGPT
   - Create appropriate storage class
   - Fix PVC and pod configuration
   - Verify storage provisioning

### Success Criteria

1. You can install and configure K8sGPT
2. You understand the difference between manual and AI-assisted debugging
3. You can use both CLI and Operator effectively
5. You can interpret and act on K8sGPT's suggestions