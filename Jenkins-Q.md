Q1. Explain your CI/CD pipeline in your current project.
Interview Answer

“In my current project, we follow a GitOps-based CI/CD pipeline using GitHub, Jenkins, Docker, Amazon ECR, Argo CD, and Amazon EKS.

1) The process starts when a developer creates a feature branch, implements the code changes, and raises a Pull Request. Before the PR is merged, mandatory code reviews are performed and automated checks such as unit tests and SonarQube quality gates are executed. Once the PR is approved and merged into the main branch, GitHub triggers a Jenkins pipeline using webhooks.”

2)“Jenkins first checks out the latest source code from GitHub and builds the application using Maven. then executes unit tests cases and generates the application artifact (JAR). Next, SonarQube performs static code analysis to detect bugs,duplicated code, and security vulnerabilities.

3)“After the quality checks, Jenkins builds a Docker image using the application’s Dockerfile. The image is then scanned using Trivy to identify any critical vulnerabilities. If critical vulnerabilities are detected, the build is failed and the image is not pushed further.”

4)“Once the image passes all security checks, Jenkins authenticates with Amazon ECR using an IAM role and pushes the versioned Docker image to the ECR repository. The pipeline then updates the Helm chart or Kubernetes deployment manifest with the new image tag and commits that change to our GitOps repository.”

5)“Argo CD continuously monitors the GitOps repository. As soon as it detects the updated image tag, it compares the desired state stored in Git with the actual state of the EKS cluster and automatically synchronizes the changes. Kubernetes then performs a rolling update by creating new pods, waiting for their startup and readiness probes to succeed, gradually routing traffic to the new version, and finally terminating the old pods. This ensures zero downtime during deployment.”

6)“After deployment, we validate the rollout by checking pod status, rollout history, application logs, ALB target health, and Grafana dashboards. If any issue is detected, Argo CD or Kubernetes allows us to quickly roll back to the previous stable version. Throughout the pipeline, notifications about build status, deployment success, or failures are sent to Microsoft Teams or Slack, ensuring the team is informed in real time.”

=======================================================

Q2. Jenkins pipeline needs a database password to deploy an application to EKS. Where would you store it and how would Jenkins retrieve it securely?

“I would not store the database password in the Jenkinsfile or Git repository.

If it’s a Jenkins-specific credential, I can store it in Jenkins Credentials Manager and reference it by credential ID.

However, if the organization uses HashiCorp Vault as the centralized secrets platform, I would prefer storing the database secret in Vault. Jenkins would authenticate to Vault using an approved authentication method, receive a short-lived Vault token with a least-privilege policy, and retrieve the secret only during the deployment stage.

The secret would be injected at runtime and never hardcoded or printed in the Jenkins console.

For an EKS application, I would also consider whether the application itself should retrieve the secret from Vault at runtime rather than passing the database password through Jenkins.”

Interviewer: “Why shouldn’t Jenkins pass the DB password to Kubernetes?”

“It can, but I would avoid unnecessarily exposing secrets through CI/CD. For workloads running on EKS, a stronger design is for the application to retrieve secrets at runtime using a workload identity and Vault integration, depending on the organization’s architecture.”

===================================================================

Q3. What is the difference between Jenkins Credentials Manager and HashiCorp Vault?

“Jenkins Credentials Manager is primarily designed to securely store and provide credentials that Jenkins jobs need, such as SSH keys, API tokens, username/password credentials and secret files.

HashiCorp Vault is a centralized enterprise secrets-management platform. It provides capabilities such as fine-grained policies, secret leasing, TTLs, dynamic credentials, rotation and audit logging.

So if I have a simple Jenkins-specific credential, Jenkins Credentials Manager may be sufficient.

If multiple systems such as Jenkins, Kubernetes applications and automation tools need centralized secret management, or if I need dynamic secrets and automated rotation, I would use Vault.

I also wouldn’t automatically put every credential into Vault. The choice depends on the organization’s security architecture and requirements.”

==================================================================

Q4. Jenkins needs to deploy infrastructure to AWS. How would you authenticate Jenkins without storing AWS access keys?

This is very important for your interviews, especially given your previous interview feedback around IAM/OIDC.

“I would avoid storing long-lived AWS access keys and secret keys in Jenkins.

I would use an identity-based authentication mechanism, preferably OIDC where supported, to establish trust between Jenkins and AWS IAM.

Jenkins authenticates using its identity, AWS STS validates the trust relationship, and STS issues temporary credentials associated with an IAM role.

That role would have only the permissions required for the pipeline, following least privilege.

The temporary credentials expire automatically, which is much safer than storing permanent access keys.”

Why not store AWS keys in Jenkins Credentials Manager?”

Answer:

“It’s possible, but long-lived access keys increase the blast radius if compromised. I prefer short-lived credentials through IAM roles and OIDC wherever the Jenkins environment and AWS integration support it.”

==================================================================

Q5. Explain Jenkins → Vault authentication and authorization.

This is probably the most important Vault question.

“First, Jenkins needs to authenticate to Vault using an authentication method configured by the organization. Depending on where Jenkins runs, this could be AppRole, Kubernetes authentication or another supported method.

Vault validates Jenkins’s identity and issues a Vault token.

That token is associated with a Vault policy. The policy determines exactly which secrets Jenkins can access.

Jenkins then uses that token to retrieve the required secret at runtime.

I would use short-lived tokens, least-privilege policies and Vault audit logging. I would also avoid putting the authentication secret itself directly into the Jenkinsfile.”

If interviewer asks:

“Authentication and authorization are the same?”

Say:

“No. Authentication establishes who Jenkins is; authorization determines what that identity is allowed to access.”

==================================================================

Q6. A developer accidentally prints a secret in Jenkins logs. What do you do?

This is a real production security scenario.

“First, I would treat the secret as compromised rather than assuming Jenkins masking makes it safe.

I would immediately identify which credential was exposed, revoke or rotate it, and determine whether the log was accessible to other users or systems.

If the credential was a Vault secret, I would revoke or rotate it according to the secret type and check Vault audit logs for suspicious access.

If it was an AWS credential, I would revoke the affected credentials and investigate CloudTrail for unauthorized activity.

Then I would remove or restrict access to the affected Jenkins build logs according to our incident procedure.

For prevention, I would review the pipeline to ensure secrets aren’t passed to commands that print them, use Jenkins credential masking appropriately, avoid shell debugging such as set -x, enforce least privilege, and add security checks/review guidelines around secret handling.”

==================================================================


Q7. Declarative vs Scripted Jenkins Pipeline


“Jenkins supports two pipeline syntaxes: Declarative and Scripted.

Declarative Pipeline uses a predefined, structured syntax. It is easier to read, maintain and validate, and is generally preferred for standard CI/CD pipelines.

Scripted Pipeline is Groovy-based and gives much more programming flexibility. I would use it when the pipeline requires complex custom logic that is difficult to express using Declarative syntax.

In production, I generally prefer Declarative Pipeline for the main pipeline structure and use script {} blocks when I need limited custom Groovy logic.”

pipeline {
    agent any

    environment {
        APP_NAME = 'payment-api'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t payment-api:${BUILD_NUMBER} .'
            }
        }

        stage('Test') {
            steps {
                sh 'pytest'
            }
        }

        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }

    post {
        success {
            echo 'Deployment successful'
        }

        failure {
            echo 'Pipeline failed'
        }
    }
}

Which one do you prefer?”

Don’t say:

“Declarative always.”

Say:

“I prefer Declarative Pipeline for standard production CI/CD because it provides a consistent structure and is easier for teams to maintain. If I have complex custom logic that isn’t cleanly represented declaratively, I can use Scripted Pipeline features or script {} blocks within a Declarative Pipeline.”

==================================================================

Q8. How do you handle credentials securely in Jenkins?

This is a high-priority question for your Mastercard interview because it connects Jenkins, AWS, Vault, IAM/OIDC and security.

“I never hardcode credentials in the Jenkinsfile, Git repository, Dockerfile, or shell scripts.

For Jenkins-specific credentials, I use Jenkins Credentials Manager and reference the credential by its ID. Jenkins injects the credential only during the required stage, and I make sure it isn’t printed in console logs.

For enterprise secret management, especially when multiple applications or environments need secrets, I would integrate Jenkins with HashiCorp Vault or the organization’s approved secrets manager. Jenkins authenticates to Vault using an appropriate identity mechanism and retrieves only the required secret at runtime using a least-privilege policy.

For AWS access, I prefer IAM roles with OIDC and short-lived STS credentials instead of storing long-lived AWS access keys in Jenkins.

I also apply RBAC, least privilege, credential rotation, short TTLs where possible, audit logging, environment separation, and secret masking.

If a secret is accidentally exposed, I treat it as compromised, immediately revoke or rotate it, investigate access logs, and fix the pipeline so it cannot happen again.”

=================================================================

Q9. Jenkins Pipeline Onboarding From Scratch

When onboarding a new application into Jenkins, I first understand the application’s build, test, deployment, infrastructure and security requirements.

I create or onboard the application’s Git repository and add a Jenkinsfile. I configure Jenkins to connect to the repository using a webhook or multibranch pipeline, so a code change automatically triggers the pipeline.

I then define the standard pipeline stages: checkout, build, unit test, code-quality and security scanning, Docker image build, container scanning, push to ECR, and deployment.

For Kubernetes applications, I normally separate CI from CD. Jenkins handles build, test and image publishing, while ArgoCD handles GitOps-based deployment to EKS.

Credentials are managed through Jenkins Credentials Manager or Vault, and AWS authentication should preferably use IAM roles/OIDC with short-lived credentials rather than static access keys.

Finally, I configure approvals for production where required, notifications, monitoring, rollback and auditability. For multiple microservices, I standardize this using Jenkins Shared Libraries and reusable templates.”

1. Requirement gathering → application, build, deployment, security and environment requirements.
2. Git repository → create/onboard repo and define branch strategy.
3. Jenkins integration → configure GitHub/GitLab webhook or Multibranch Pipeline.
4. Jenkinsfile → create pipeline using Declarative Pipeline/Shared Library.
5. Credentials → configure Jenkins Credentials Manager/Vault and secure AWS authentication.
6. CI pipeline → checkout → build → unit test → SAST/dependency scan.
7. Docker → build image → container scan → push immutable image to ECR.
8. GitOps/CD → update Helm/GitOps repository with the new image version.
9. ArgoCD/EKS → ArgoCD syncs the application to the target Kubernetes environment.
10. Production readiness → configure probes, resources, HPA, PDB, IAM, networking and approvals.
11. Post-deployment → validate health, monitoring, SLO/SLI, alerts and rollback strategy.

=================================================================

Q10. When onboarding a new microservice, I follow a standardized platform onboarding process.

First, I understand the application’s requirements: traffic, ports, dependencies, database, availability requirements, SLOs, security requirements and environment requirements.

Then I create or reuse the standard repository structure, Dockerfile, Jenkins pipeline and Kubernetes Helm/manifests.

For AWS, I provision required infrastructure through Terraform, such as IAM roles, ECR, networking or supporting services. I avoid creating infrastructure manually.

For Kubernetes, I configure Deployment, Service, readiness/liveness/startup probes, resource requests and limits, HPA, PDB and appropriate affinity or topology rules.

I integrate the service with Jenkins for CI, security scanning and image build, push the image to ECR, and use ArgoCD/GitOps for deployment to EKS.

I then configure Prometheus/Grafana monitoring, alerts and SLO/SLI dashboards.

Before production, I validate security, connectivity, scaling, failure scenarios and rollback. Finally, I document the service, ownership, dependencies and runbook and hand it over to the on-call team.”

 













