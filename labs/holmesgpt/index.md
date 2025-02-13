### HolmesGPT Lab: AI-Powered Alert Investigation

#### Prerequisites

Start minikube with 6GB memory.

Note: If you have resources deployed from a previous lab, run `minikube delete` to start with a fresh cluster.

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

4. Complete the following test scenarios:

   **Scenario 1: Fixing Resource Configuration Issues**
   - Deploy a pod with excessive resource requests
   - Ask HolmesGPT to analyze the resource configuration and provide a fixed YAML
   - Apply the fixed YAML suggested by HolmesGPT
   - Verify the fix worked, if not use HolmesGPT to debug further

   **Scenario 2: Fixing Memory Leak Issues**
   - Deploy a pod with potential memory leak
   - Ask HolmesGPT to analyze the pod and provide a fixed YAML with proper resource limits
   - Apply the fixed YAML suggested by HolmesGPT
   - Verify the fix worked, if not use HolmesGPT to debug further

   **Scenario 3: Fixing Liveness Probe Issues**
   - Deploy a pod with problematic liveness probe
   - Ask HolmesGPT to analyze the probe configuration and provide a fixed YAML
   - Apply the fixed YAML suggested by HolmesGPT
   - Verify the fix worked, if not use HolmesGPT to debug further

   **Scenario 4: Fixing Service Endpoint Issues**
   - Deploy a service with missing endpoints
   - Ask HolmesGPT to analyze the service and provide a fixed YAML including necessary deployment
   - Apply the fixed YAML suggested by HolmesGPT
   - Verify the fix worked, if not use HolmesGPT to debug further

   **Scenario 5: Fixing Security Context Issues**
   - Deploy a pod with security issues
   - Ask HolmesGPT to analyze the security context and provide a fixed YAML following best practices
   - Apply the fixed YAML suggested by HolmesGPT
   - Verify the fix worked, if not use HolmesGPT to debug further

   **Scenario 6: Fixing Pod Affinity Issues**
   - Deploy pods with problematic affinity rules
   - Ask HolmesGPT to analyze the affinity rules and provide a fixed YAML
   - Apply the fixed YAML suggested by HolmesGPT
   - Verify the fix worked, if not use HolmesGPT to debug further

   **Scenario 7: Fixing Network Policy Issues**
   - Deploy a problematic network policy
   - Ask HolmesGPT to analyze the policy and provide a fixed YAML that allows essential traffic
   - Apply the fixed YAML suggested by HolmesGPT
   - Verify the fix worked, if not use HolmesGPT to debug further

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
5. You can successfully:
   - Deploy problematic Kubernetes configurations
   - Use HolmesGPT to analyze issues and get fixed YAML
   - Apply the suggested YAML fixes
   - Verify if the fixes worked
   - Use HolmesGPT to debug any remaining issues
6. You have completed all test scenarios:
   - Fixed resource configuration issues
   - Fixed memory leak issues
   - Fixed liveness probe issues
   - Fixed service endpoint issues
   - Fixed security context issues
   - Fixed pod affinity issues
   - Fixed network policy issues
7. You understand how to:
   - Ask HolmesGPT for specific YAML fixes
   - Interpret and apply the suggested configurations
   - Verify the effectiveness of applied fixes
   - Debug issues when fixes don't work as expected