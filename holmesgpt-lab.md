### Part 1: Installing HolmesGPT with Helm

#### Introduction

##### **Overview of the Lab Objectives**
- Install Helm and HolmesGPT
- Configure different LLM backends
- Test different investigation scenarios
- Understand security and configuration options

##### **Brief on HolmesGPT**
- **HolmesGPT**: An AI-powered tool for investigating and resolving Kubernetes alerts
- **Key Features**:
  - Automatic data collection
  - Read-only access with RBAC permissions
  - Runbook automation
  - Multiple LLM backend support
  - Integration with existing tools (PagerDuty, etc.)

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
   - Deploy a pod with resource issues
   - Deploy a pod with image issues
   - What issues do you expect HolmesGPT to find?

2. Install and configure HolmesGPT CLI
   - Install pipx if needed
   - Install HolmesGPT CLI
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

#### Challenge Tasks

1. Add more alert types to your runbooks
   - What other alerts might be useful?
   - How would you structure the investigation steps?

2. Test different LLM backends
   - Try different models
   - Compare response quality

3. Create a runbook for a complex scenario
   - Combine multiple issues
   - Include conditional investigation steps

### Success Criteria

1. You have HolmesGPT installed and configured
2. You can run basic investigations
3. You have created and tested custom runbooks
4. You understand how to extend runbooks for new scenarios

### Additional Resources
- HolmesGPT Documentation
- Prometheus Operator Documentation
- AlertManager Documentation