### HolmesGPT Lab: AI-Powered Alert Investigation

#### Prerequisites

Start minikube with 6GB memory.

#### Introduction

##### **Overview of the Lab Objectives**
- Install Helm and HolmesGPT
- Test different investigation scenarios
- Understand security and configuration options

For detailed documentation, see: https://github.com/robusta-dev/holmesgpt

##### **Brief on HolmesGPT**
- **HolmesGPT**: An AI-powered tool for investigating and resolving Kubernetes alerts
- **Key Features**:
  - Automatic data collection
  - Read-only access with RBAC permissions
  - Runbook automation
  - Multiple LLM backend support
  - Integration with existing tools (Prometheus, AlertManager, etc.)

#### Lab Tasks

1. Install and configure Helm
   - Install Helm on your system
   - Verify the installation

2. Create necessary configuration files for HolmesGPT
   - Create a values file for Helm configuration
   - Set up API key secrets
   - Add and update Helm repositories

3. Install HolmesGPT using Helm
   - Install HolmesGPT with your configuration
   - Verify the installation

### Part 2: Testing HolmesGPT

1. Create test scenarios
   - Deploy a pod that requests more memory than available (1000Gi)
   - Deploy a pod with a non-existent image tag (nginx:nonexistent)
   - What alerts do you expect these issues to generate?

2. Install and configure HolmesGPT CLI
   - Install pipx if needed
   - Install HolmesGPT CLI (v0.7.2)
   - Configure your API key

3. Test basic HolmesGPT commands
   - Ask about unhealthy pods
   - Check exposed services
   - Investigate specific resources

### Part 3: Custom Runbooks with AlertManager

1. Set up monitoring infrastructure
   - Install Prometheus stack
   - Deploy a test application
   - Create AlertManager rules

2. Create custom runbooks
   - Run initial investigation without runbooks
   - Create runbooks for common issues
   - Test runbooks with HolmesGPT

#### Challenge Tasks (Optional)

1. Add more alert types to your runbooks
   - What other alerts might be useful?
   - How would you structure the investigation steps?

2. Customize investigation depth
   - Try different verbosity levels
   - Test focused vs broad investigations

3. Create a runbook for a complex scenario
   - Combine multiple issues
   - Include conditional investigation steps

### Success Criteria

1. You have HolmesGPT installed and configured
2. You can run basic investigations
3. You have created and tested custom runbooks
4. You understand how to extend runbooks for new scenarios