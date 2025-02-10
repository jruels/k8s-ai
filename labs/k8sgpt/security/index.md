### K8sGPT Lab: AI-Powered Security Scanning

#### Prerequisites

Start minikube with 6GB memory:
```bash
minikube start --memory=6144
```

Note: If you have resources deployed from a previous lab, run `minikube delete` to start with a fresh cluster.

#### Introduction

##### **Overview of Lab Objectives**
- Enable and configure K8sGPT security scanning
- Create and analyze security issues
- Fix common security misconfigurations
- Verify security fixes

For detailed documentation, see: https://docs.k8sgpt.ai/

##### **Brief on K8sGPT Security Features**
- **Trivy Integration**: Scans for vulnerabilities and misconfigurations
- **Key Capabilities**:
  - Configuration auditing
  - Security best practices validation
  - Remediation suggestions
  - Real-time security monitoring

#### Lab Tasks

### Part 1: Enable Security Scanning

1. Enable Trivy integration
   - Check available integrations
   - Activate Trivy scanner
   - Verify integration status

2. Test security scanning
   - Run initial configuration audit
   - Check for vulnerabilities
   - Review scanning results

### Part 2: Security Scenarios

1. Privileged Container
   - Create a pod with privileged access
   - Analyze the security implications
   - Apply security best practices
   - Verify the fix

2. Writable Root Filesystem
   - Deploy a pod with writable root filesystem
   - Identify the security risks
   - Implement read-only root filesystem
   - Confirm the configuration

3. Host Network Access
   - Set up a pod with host network access
   - Understand the security concerns
   - Apply network isolation
   - Validate the changes

4. Host Path Volume
   - Create a pod with host path volume
   - Analyze volume security
   - Implement secure volume options
   - Check the results

5. Service Account Token Mounting
   - Deploy a pod with default token mounting
   - Review token security
   - Configure secure token handling
   - Verify token settings

### Success Criteria

1. You can enable and configure security scanning
2. You understand common security misconfigurations
3. You can identify security issues using K8sGPT
4. You can implement security fixes
5. You can verify security improvements

### Tips

- Use `k8sgpt analyze --filter ConfigAuditReport` to check configurations
- Use jq to filter specific security issues:
  ```bash
  # Example structure for filtering issues
  k8sgpt analyze --filter ConfigAuditReport --explain --output json | \
    jq '.results[] | select(.name == "default/your-pod-name") | \
    select(.error[].Text | contains("security-text-to-find"))'
  ```
- Review the full details of security findings
- Test fixes before applying to production
- Consider security context settings
- Understand the principle of least privilege