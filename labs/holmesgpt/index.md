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

   **Scenario 1: Resource Configuration Issues**
   - Deploy a pod with excessive resource requests
   - Use HolmesGPT to analyze and fix the configuration

   **Scenario 2: Memory Leak Issues**
   - Deploy a pod with potential memory leak
   - Add proper resource limits and monitoring

   **Scenario 3: Liveness Probe Issues**
   - Deploy a pod with problematic liveness probe
   - Fix probe configuration and add readiness probe

   **Scenario 4: Service Endpoint Issues**
   - Deploy a service with missing endpoints
   - Fix service and deployment configuration

   **Scenario 5: Security Context Issues**
   - Deploy a pod with security issues
   - Implement security best practices

   **Scenario 6: Pod Affinity Issues**
   - Deploy pods with problematic affinity rules
   - Fix node affinity configuration

   **Scenario 7: Network Policy Issues**
   - Deploy a problematic network policy
   - Configure appropriate traffic rules

5. Using HolmesGPT for Complex Analysis

   **Security Analysis**
   - Check for security vulnerabilities
   - Analyze network policies
   - Check RBAC configuration

   **Performance Investigation**
   - Analyze resource utilization
   - Check node status
   - Review scaling patterns

   **Configuration Audit**
   - Check for best practices
   - Analyze resource quotas
   - Review storage configuration

6. Troubleshooting Common Issues

   **Investigating CrashLoopBackOff**
   - Get detailed analysis of crashing pods
   - Check container logs

   **Network Connectivity Issues**
   - Test service connectivity
   - Check DNS resolution

   **Resource Constraints**
   - Analyze node pressure
   - Check pod evictions

7. Integration with Other Tools

   **Using with kubectl**
   - Analyze pod lists
   - Review cluster events

   **Using with Logs**
   - Analyze application logs
   - Review system logs

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