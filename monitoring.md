1) **"Dynatrace is an AI-powered observability** platform that provides monitoring for applications, infrastructure, Kubernetes,
   logs, and user experience. It uses OneAgent for automatic data collection and Davis AI for root cause analysis.
    In production environments, Dynatrace helps us quickly identify performance bottlenecks, reduce MTTR, monitor microservices, and improve application reliability."

**Difference from Splunk (one-line interview answer):**

Splunk is primarily used for log analytics, while Dynatrace provides full-stack observability including APM, 
infrastructure monitoring, tracing, logs, and AI-driven root cause analysis.


2) **"SonarQube is a static code analysis** tool used to measure code quality and security.
      We integrate it into CI/CD pipelines so every code commit is automatically scanned for bugs,
      vulnerabilities, code smells, and test coverage. We use Quality Gates to stop deployments when code quality standards are not met.
      This helps reduce technical debt and improves application reliability.

4) **"We use JFrog Artifactory as a centralized artifact repository**. After the CI pipeline builds the application,
      the generated artifacts such as JARs or Docker images are uploaded to Artifactory with version tagging.
      During release management, we promote the same validated artifact across DEV, TEST, UAT, and PROD instead of rebuilding it.
      This ensures artifact traceability, rollback capability, consistency across environments, and better governance of software releases.
