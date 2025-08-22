NIRMATA GO-AGENT-REMEDIATOR COMPLETE INSTALLATION GUIDE
======================================================

This guide provides step-by-step instructions for installing and configuring the Nirmata go-agent-remediator system. The system automatically detects Kubernetes policy violations, analyzes them using AI, and creates GitHub pull requests with intelligent fixes.

SYSTEM OVERVIEW
---------------
The remediator system consists of these components:
- Kyverno N4K: Detects policy violations in Kubernetes
- Reports Server: Stores violation data
- Remediator Agent: AI-powered remediation engine
- AWS Bedrock: Large Language Model for generating fixes
- GitHub Integration: Creates pull requests with fixes
- ArgoCD: Manages deployments via GitOps

PREREQUISITES
-------------
Before starting, ensure you have:
1. EKS Kubernetes cluster (minimum 3 nodes, t3a.medium or larger)
2. ArgoCD installed and running
3. AWS CLI configured with proper permissions
4. kubectl configured to access your cluster
5. GitHub account with repository access
6. GitHub CLI (gh) installed and authenticated
7. AWS Bedrock access enabled (we will configure this)

ESTIMATED INSTALLATION TIME
---------------------------
- Initial setup: 30-45 minutes
- Troubleshooting (if needed): 15-30 minutes
- End-to-end testing: 10-15 minutes
Total: 1-1.5 hours

STEP 1: CLUSTER PREPARATION
---------------------------

1.1 Verify cluster access
Run this command to ensure you can access your cluster:
kubectl get nodes

You should see 3 or more nodes in Ready status.

1.2 Install EKS Pod Identity Add-on
The remediator needs to access AWS Bedrock using Pod Identity. Install the add-on:

aws eks create-addon --cluster-name YOUR_CLUSTER_NAME --addon-name eks-pod-identity-agent --resolve-conflicts OVERWRITE --profile YOUR_PROFILE --region YOUR_REGION

Replace YOUR_CLUSTER_NAME, YOUR_PROFILE, and YOUR_REGION with your actual values.

Wait 2-3 minutes for the add-on to install, then verify:
kubectl get pods -n kube-system | grep pod-identity

You should see pod-identity-agent pods running on each node.

Optional:
1.3 Optimize ArgoCD for fast syncing
By default, ArgoCD syncs every 3-5 minutes. We will configure it for 30-second syncing:

kubectl patch configmap argocd-cm -n argocd --patch='{"data":{"timeout.reconciliation":"30s","timeout.hard.reconciliation":"30s","application.resync":"30s","server.repo.server.timeout.seconds":"60"}}'

kubectl patch configmap argocd-cmd-params-cm -n argocd --patch='{"data":{"application.sync.retry.duration":"30s","reposerver.parallelism.limit":"10"}}'

Restart ArgoCD components to apply changes:
kubectl rollout restart deployment/argocd-repo-server -n argocd
kubectl rollout restart deployment/argocd-server -n argocd  
kubectl rollout restart statefulset/argocd-application-controller -n argocd

STEP 2: GITHUB REPOSITORY SETUP
--------------------------------

2.1 Create your GitOps repository
Create a new GitHub repository for this project:
gh repo create project-remediator-agent --public

Clone it locally:
git clone https://github.com/YOUR_USERNAME/project-remediator-agent.git
cd project-remediator-agent

2.2 Create directory structure
mkdir -p remediator-agent applications test-workloads

2.3 Create GitHub access token
Generate a GitHub token with repo permissions:
gh auth token

Copy this token - you will need it later for Kubernetes secret.

STEP 3: AWS BEDROCK AND IAM SETUP
----------------------------------

3.1 Choose your LLM model
List available models in your region:
aws bedrock list-inference-profiles --region us-west-2 --query 'inferenceProfileSummaries[*].[inferenceProfileId,inferenceProfileName]' --output table --profile YOUR_PROFILE

For this guide, we recommend: us.anthropic.claude-sonnet-4-20250514-v1:0
This provides excellent reasoning for security policy analysis.

3.2 Get model ARNs for permissions
Get the inference profile details:
aws bedrock list-inference-profiles --region us-west-2 --query 'inferenceProfileSummaries[?inferenceProfileId==`us.anthropic.claude-sonnet-4-20250514-v1:0`]' --profile YOUR_PROFILE

Note down the foundation model ARNs from all regions (usually us-east-1, us-east-2, us-west-2).

3.3 Create IAM trust policy
Create file simple-trust-policy.json:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}

3.4 Create IAM role
aws iam create-role --role-name remediator-agent-role --assume-role-policy-document file://simple-trust-policy.json --profile YOUR_PROFILE

3.5 Create Bedrock access policy
Create file bedrock-policy.json (replace with your actual ARNs from step 3.2):
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0",
        "arn:aws:bedrock:us-east-2::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0", 
        "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0",
        "arn:aws:bedrock:us-west-2:YOUR_ACCOUNT_ID:inference-profile/us.anthropic.claude-sonnet-4-20250514-v1:0"
      ]
    }
  ]
}

Apply the policy:
aws iam put-role-policy --role-name remediator-agent-role --policy-name BedrockInvokePolicy --policy-document file://bedrock-policy.json --profile YOUR_PROFILE

3.6 Create Pod Identity association
Get your account ID first:
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --profile YOUR_PROFILE --query 'Account' --output text)

Create the association:
aws eks create-pod-identity-association --cluster-name YOUR_CLUSTER_NAME --role-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/remediator-agent-role --namespace nirmata --service-account remediator-agent --profile YOUR_PROFILE --region YOUR_REGION

STEP 4: KUBERNETES SECRETS SETUP
---------------------------------

4.1 Create namespaces
kubectl create namespace nirmata
kubectl create namespace kyverno

4.2 Create GitHub secret
Use the token from step 2.3:
kubectl create secret generic github-token --from-literal=token=YOUR_GITHUB_TOKEN --namespace nirmata

STEP 5: INSTALL KYVERNO AND REPORTS SERVER
-------------------------------------------

5.1 Install Kyverno using Helm
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install n4k-kyverno kyverno/kyverno --namespace kyverno --create-namespace --set features.admissionReports.enabled=true --set features.backgroundReports.enabled=true --set features.policyReports.enabled=true

5.2 Install Reports Server using Helm
helm repo add nirmata https://nirmata.github.io/kyverno-charts
helm repo update

helm install reports-server nirmata/reports-server --namespace kyverno --set etcd.enabled=true

5.3 Install baseline policies using Kustomize
Clone the policies repository and apply baseline policies:
git clone https://github.com/kyverno/policies.git kyverno-policies
kubectl apply -k kyverno-policies/pod-security/baseline/

5.4 Verify installation
Wait 2-3 minutes then check:
kubectl get pods -n kyverno
kubectl get cpol

All pods should be Running and all policies should show ADMISSION=true, BACKGROUND=true.

STEP 6: CONFIGURE REMEDIATOR AGENT
-----------------------------------

6.1 Create remediator configuration
Create file remediator-agent/custom-values.yaml:

replicaCount: 1

image:
  repository: ghcr.io/nirmata/go-agent-remediator
  tag: "0.2.0-rc4"
  pullPolicy: IfNotPresent

serviceAccount:
  create: true
  name: "remediator-agent"

resources:
  limits:
    memory: 1Gi
  requests:
    cpu: 200m
    memory: 256Mi

llm:
  enabled: true
  provider: bedrock
  model: "us.anthropic.claude-sonnet-4-20250514-v1:0"
  region: "us-west-2"

tool:
  enabled: true
  name: github-tool
  type: github
  secretName: "github-token"
  secretKey: "token"
  defaults:
    prBranchPrefix: "fix/policy-violation-"
    prTitleTemplate: "fix: Policy violation remediation in {{.File}}"
    commitMessageTemplate: "fix: Remediate {{.PolicyName}} policy violation in {{.File}}"

remediator:
  enabled: true
  name: remediator-agent
  environment:
    localCluster: false
    argoCD:
      hub: true
  target:
    argoAppSelector:
      allApps: false
      names: 
        - test-violations
  remediation:
    schedule: "*/5 * * * *"    # note: change remediation time to hours after testing
    actions:
      - type: CreateGithubPR
        toolRefName: github-tool

6.2 Create ArgoCD application for remediator
Create file applications/remediator-agent.yaml:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: remediator-agent
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://nirmata.github.io/kyverno-charts
      chart: remediator-agent
      targetRevision: 0.2.0-rc4
      helm:
        valueFiles:
          - $values/remediator-agent/custom-values.yaml
    - repoURL: https://github.com/YOUR_USERNAME/project-remediator-agent.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: nirmata
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

Replace YOUR_USERNAME with your GitHub username.

git add remediator-agent/ applications/
git commit -m "Add remediator agent configuration"
git push

kubectl apply -f applications/remediator-agent.yaml

6.4 Verify remediator deployment
kubectl get pods -n nirmata
kubectl logs -n nirmata deployment/remediator-agent --tail=50

The log should show successful startup and ArgoCD discovery.

STEP 7: CREATE TEST VIOLATIONS
-------------------------------

7.1 Create test violation with clear documentation
Create file test-workloads/privileged-deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: privileged-test
  namespace: application
  annotations:
    # POLICY VIOLATION: This deployment violates security policies
    # ISSUE: Container runs with privileged=true which grants host-level access
    # POLICY: Kyverno disallow-privileged-containers policy
    # EXPECTED FIX: Set privileged=false or remove privileged flag
spec:
  replicas: 1
  selector:
    matchLabels:
      app: privileged-test
  template:
    metadata:
      labels:
        app: privileged-test
    spec:
      containers:
      - name: test-container
        image: nginx:latest
        securityContext:
          privileged: true  # VIOLATION: This should be false or removed
        ports:
        - containerPort: 80

7.2 Create ArgoCD application for test violations
Create file applications/test-violations.yaml:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test-violations
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/project-remediator-agent.git
    path: test-workloads
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: application
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: true
    syncOptions:
      - CreateNamespace=true
      - Replace=true
      - Force=true
    retry:
      limit: 10
      backoff:
        duration: 30s
        maxDuration: 5m

Replace YOUR_USERNAME with your GitHub username.

7.3 Deploy test violation
git add test-workloads/ applications/test-violations.yaml
git commit -m "Add test violation deployment"
git push

kubectl apply -f applications/test-violations.yaml

STEP 8: VERIFICATION AND TESTING
---------------------------------

8.1 Watch ArgoCD sync (should happen in 30 seconds)
kubectl get applications -n argocd -w

Wait for test-violations to show Synced status.

8.2 Verify violation deployment
kubectl get pods -n application
kubectl get deployments -n application

8.3 Check policy violation detection
kubectl get policyreports -n application
kubectl get policyreports -n application -o yaml

You should see a policy report showing the privileged container violation.

8.4 Monitor remediator logs
kubectl logs -n nirmata deployment/remediator-agent --tail=50 -f

Look for messages like:
- "Final selected applications: 1"
- "Fetching policy violations for application: test-violations"
- "Building violation context"
- "Creating GitHub PR"

8.5 Check for GitHub PR creation
Within 2-5 minutes, check for pull requests:
gh pr list --repo YOUR_USERNAME/project-remediator-agent

You should see a new PR with an intelligent fix for the privileged container violation.

EXPECTED RESULTS
----------------

Successful operation shows:
1. ArgoCD syncs changes in 30 seconds (not 5 minutes)
2. Kyverno detects policy violations immediately  
3. Remediator processes violations every 2 minutes
4. AWS Bedrock provides intelligent analysis
5. GitHub PRs are created with proper fixes
6. End-to-end cycle completes in under 3 minutes

TROUBLESHOOTING GUIDE
---------------------

Issue 1: Pod Identity timeout errors
Symptom: "failed to refresh cached credentials, get identity timeout"
Solution: Verify eks-pod-identity-agent addon is installed and running:
kubectl get pods -n kube-system | grep pod-identity
If missing, reinstall the addon (step 1.2).

Issue 2: Bedrock access denied
Symptom: "AccessDeniedException: User is not authorized to perform bedrock:InvokeModel"  
Solution: Verify IAM policy includes ALL foundation model ARNs from all regions.
The inference profile routes dynamically between regions.

Issue 3: GitHub API rate limit
Symptom: "403 API rate limit exceeded"
Solution: Ensure argoAppSelector is configured to monitor specific apps:
Set allApps: false and names: [test-violations] in custom-values.yaml

Issue 4: ArgoCD slow sync (5+ minutes)
Symptom: Changes take 5+ minutes to deploy
Solution: Apply the ArgoCD optimization from step 1.3 and restart components.

Issue 5: GitHub PR creation fails
Symptom: "GET .../git/ref/heads/HEAD: 404 Not Found"
Solution: Ensure targetRevision is set to "main" (not "HEAD") in application YAML files.

Issue 6: Application not syncing after changes  
Symptom: ArgoCD shows "Synced" but no changes deployed
Solution: Use kubectl to force sync:
kubectl patch application test-violations -n argocd --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'

INDIVIDUAL VIOLATION TESTING
-----------------------------

For testing multiple violation types individually:

1. Create additional violation files locally (do not commit):
- capabilities-deployment.yaml (dangerous capabilities)
- hostpath-deployment.yaml (host filesystem access)  
- hostports-deployment.yaml (host port binding)
- hostnamespaces-deployment.yaml (host namespace sharing)

2. Test one violation at a time:
git add test-workloads/capabilities-deployment.yaml
git commit -m "test: Add capabilities violation"  
git push

3. Wait for PR creation, then clean up:
git rm test-workloads/capabilities-deployment.yaml
git commit -m "clean: Remove test violation"
git push

4. Repeat for each violation type

This approach allows systematic testing of different policy violations.

ADVANCED CONFIGURATION
-----------------------

Custom LLM Models:
To use different Bedrock models, update the model field in custom-values.yaml:
- For Claude 3.5 Sonnet: "anthropic.claude-3-5-sonnet-20241022-v2:0"
- For Nova Premier: "us.amazon.nova-premier-v1:0"  
- For Llama 3.2: "us.meta.llama3-2-90b-instruct-v1:0"

Custom Scheduling:
Change remediation frequency in custom-values.yaml:
schedule: "*/5 * * * *"  # Every 5 minute (for testing)
schedule:  "0 */6 * * *"  # For 6 hours for stable environments

Multiple Applications:
Monitor multiple ArgoCD applications:
argoAppSelector:
  allApps: false
  names:
    - test-violations
    - staging-app
    - production-app

PRODUCTION RECOMMENDATIONS
--------------------------

1. Security:
- Use least-privilege IAM policies
- Enable AWS CloudTrail for Bedrock API calls
- Rotate GitHub tokens regularly
- Use private GitHub repositories

2. Performance:
- Scale cluster based on workload
- Monitor Bedrock API usage and costs
- Set appropriate resource limits on remediator

3. Operations:
- Set up monitoring for remediator logs
- Configure ArgoCD notifications for failures
- Establish PR review processes for auto-fixes
- Test remediation logic in non-production first

4. Cost Optimization:
- Use smaller models for simple violations
- Adjust remediation frequency based on needs
- Monitor AWS Bedrock costs in billing console

SUPPORT AND RESOURCES
----------------------

Official Documentation:
- Nirmata Kyverno: https://docs.nirmata.io/
- Kyverno Policies: https://kyverno.io/policies/
- AWS Bedrock: https://docs.aws.amazon.com/bedrock/

Troubleshooting:
- Check remediator logs: kubectl logs -n nirmata deployment/remediator-agent
- Verify ArgoCD status: kubectl get applications -n argocd
- Monitor Kyverno: kubectl get cpol and kubectl get policyreports -A

This guide represents a complete, tested installation process that addresses all common issues and provides a working end-to-end automated policy remediation system.