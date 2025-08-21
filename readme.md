GO-AGENT-REMEDIATOR INSTALLATION GUIDE
=====================================

OVERVIEW
--------
This guide covers the complete installation of the Nirmata go-agent-remediator system 
for automated Kyverno policy violation remediation using AWS Bedrock LLM and GitHub PRs.

PREREQUISITES
-------------
1. EKS cluster with minimum 3 nodes (t3a.medium or larger)
2. ArgoCD installed and operational
3. AWS CLI configured with appropriate permissions
4. kubectl configured for cluster access
5. Helm 3.x installed
6. GitHub account with repository access
7. AWS Bedrock access in us-west-2 region

ARCHITECTURE COMPONENTS
-----------------------
- N4K Kyverno: Policy engine and violation detection
- Reports Server: Policy violation storage (etcd backend)
- Remediator Agent: LLM-powered remediation engine
- AWS Bedrock: LLM inference for generating fixes
- GitHub Integration: Pull request creation for fixes
- ArgoCD: GitOps application management

STEP 1: INFRASTRUCTURE SETUP
-----------------------------

1.1 Scale EKS Cluster (if needed)
Get current node group name:
aws eks list-nodegroups --cluster-name YOUR_CLUSTER_NAME --profile YOUR_PROFILE --region YOUR_REGION

Scale to 3 nodes:
aws eks update-nodegroup-config --cluster-name YOUR_CLUSTER_NAME --nodegroup-name YOUR_NODEGROUP_NAME --scaling-config desiredSize=3 --profile YOUR_PROFILE --region YOUR_REGION

1.2 Install AWS EBS CSI Driver (if missing)
aws eks create-addon --cluster-name YOUR_CLUSTER_NAME --addon-name aws-ebs-csi-driver --resolve-conflicts OVERWRITE --profile YOUR_PROFILE --region YOUR_REGION

1.3 Create GitOps Repository
Create new GitHub repository for GitOps configuration
Clone repository locally
mkdir -p charts applications test-workloads

STEP 2: DEPLOY KYVERNO AND REPORTS SERVER
------------------------------------------

2.1 Add Helm Repository
helm repo add nirmata https://nirmata.github.io/kyverno-charts
helm repo update

2.2 Pull Required Charts
helm pull nirmata/kyverno --version 3.4.7 --untar --untardir charts/
helm pull nirmata/reports-server --version 0.2.6 --untar --untardir charts/

2.3 Create ArgoCD Applications

Create applications/kyverno.yaml:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: n4k-kyverno
  namespace: argocd
  labels:
    app.kubernetes.io/name: n4k-kyverno
    app.kubernetes.io/part-of: nirmata-platform
    environment: test
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/YOUR_REPO.git
    path: charts/kyverno
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: kyverno
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ServerSideApply=true
      - Replace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10

Create applications/reports-server.yaml:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: reports-server
  namespace: argocd
  labels:
    app.kubernetes.io/name: reports-server
    app.kubernetes.io/part-of: nirmata-platform
    environment: test
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/YOUR_REPO.git
    path: charts/reports-server
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: kyverno
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10

2.4 Disable Migration Hook in Kyverno
Edit charts/kyverno/values.yaml:
Find the migration section (around line 129) and change:
  migration:
    enabled: false

2.5 Deploy Applications
git add charts/ applications/
git commit -m "Add Kyverno and reports-server charts and applications"
git push

kubectl apply -f applications/reports-server.yaml
kubectl apply -f applications/kyverno.yaml

2.6 Verify Deployment
kubectl get pods -n kyverno
kubectl get applications -n argocd

Wait for all pods to be Running and applications to show Synced/Healthy status.

STEP 3: DEPLOY BASELINE POLICIES
---------------------------------

3.1 Clone Policy Repository
git clone https://github.com/nirmata/kyverno-policies.git

3.2 Apply Baseline Policies
cd kyverno-policies/pod-security
kubectl apply -k baseline/
cd ../../

3.3 Verify Policies
kubectl get cpol
All policies should show Ready status with ADMISSION=true and BACKGROUND=true.

STEP 4: CREATE TEST WORKLOADS
------------------------------

4.1 Create Test Workloads Directory
mkdir -p test-workloads

4.2 Create Violating Workloads
Create test-workloads/privileged-deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: privileged-test
  labels:
    app: privileged-test
    violation: disallow-privileged-containers
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
      - name: privileged-container
        image: nginx:1.20
        securityContext:
          privileged: true
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi

Create similar files for other violations:
- hostpath-deployment.yaml (violates disallow-host-path)
- hostports-deployment.yaml (violates disallow-host-ports) 
- capabilities-deployment.yaml (violates disallow-capabilities)
- hostnamespaces-deployment.yaml (violates disallow-host-namespaces)

4.3 Create ArgoCD Application for Test Workloads
Create applications/test-violations.yaml:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test-violations
  namespace: argocd
  labels:
    app.kubernetes.io/name: test-violations
    app.kubernetes.io/part-of: remediator-testing
    environment: test
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/YOUR_REPO.git
    path: test-workloads
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: application
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10

4.4 Deploy Test Workloads
git add test-workloads/ applications/test-violations.yaml
git commit -m "Add test workloads that violate baseline pod security policies"
git push

kubectl apply -f applications/test-violations.yaml

4.5 Verify Policy Violations
kubectl get pods -n application
kubectl get policyreports -n application

Each workload should show policy violations (11 PASS, 1 FAIL per workload).

STEP 5: AUTHENTICATION SETUP
-----------------------------

5.1 Create GitHub Secret
Get GitHub token with repo permissions:
gh auth token

Create Kubernetes secret:
kubectl create namespace nirmata
kubectl create secret generic github-token --from-literal=token=YOUR_GITHUB_TOKEN --namespace nirmata

5.2 Create IAM Role for Pod Identity
Create trust policy file (simple-trust-policy.json):
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

Create IAM role:
aws iam create-role --role-name remediator-agent-role --assume-role-policy-document file://simple-trust-policy.json --description "IAM role for EKS remediator agent with Pod Identity" --profile YOUR_PROFILE

5.3 Attach Bedrock Policy
Get your AWS account ID:
aws sts get-caller-identity --profile YOUR_PROFILE --query 'Account' --output text

Create Bedrock policy (replace ACCOUNT_ID and INFERENCE_PROFILE_ID):
aws iam put-role-policy --role-name remediator-agent-role --policy-name BedrockInvokePolicy --policy-document '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BedrockInvoke",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "arn:aws:bedrock:us-west-2:ACCOUNT_ID:application-inference-profile/INFERENCE_PROFILE_ID"
    }
  ]
}' --profile YOUR_PROFILE

5.4 Create Pod Identity Association
aws eks create-pod-identity-association --cluster-name YOUR_CLUSTER_NAME --role-arn arn:aws:iam::YOUR_ACCOUNT_ID:role/remediator-agent-role --namespace nirmata --service-account remediator-agent --profile YOUR_PROFILE --region YOUR_REGION

STEP 6: DEPLOY REMEDIATOR AGENT
--------------------------------

6.1 Create Custom Values File
Create remediator-agent/custom-values.yaml:
replicaCount: 1

image:
  repository: ghcr.io/nirmata/go-agent-remediator
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
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
  model: "arn:aws:bedrock:us-west-2:YOUR_ACCOUNT_ID:application-inference-profile/YOUR_INFERENCE_PROFILE_ID"
  region: "us-west-2"
  endpoint: ""
  deploymentName: ""
  secretName: ""
  secretKey: "aws_access_key_id"

tool:
  enabled: true
  name: github-tool
  type: github
  secretName: "github-token"
  secretKey: "token"
  defaults:
    prBranchPrefix: remediation-
    prTitleTemplate: "[Auto-Remediation] Fix policy violations"
    commitMessageTemplate: "Auto-fix: Remediate policy violations"

remediator:
  enabled: true
  name: remediator-agent
  environment:
    localCluster: false
    argoCD:
      hub: true
  target:
    clusterNames: []
    clusterServerUrls: []
    argoAppSelector:
      allApps: true
  remediation:
    schedule: "*/5 * * * *"
    actions:
      - type: CreateGithubPR
        toolRefName: github-tool

nodeSelector: {}
tolerations: []
affinity: {}

6.2 Create ArgoCD Application for Remediator
Create applications/remediator-agent.yaml:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: remediator-agent
  namespace: argocd
  labels:
    app.kubernetes.io/name: remediator-agent
    app.kubernetes.io/part-of: nirmata-platform
    environment: test
spec:
  project: default
  sources:
    - repoURL: https://nirmata.github.io/kyverno-charts
      chart: remediator-agent
      targetRevision: 0.2.0-rc4
      helm:
        valueFiles:
          - $values/remediator-agent/custom-values.yaml
    - repoURL: https://github.com/YOUR_USERNAME/YOUR_REPO.git
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: nirmata
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10

6.3 Deploy Remediator Agent
git add remediator-agent/ applications/remediator-agent.yaml
git commit -m "Add remediator agent configuration and ArgoCD application"
git push

kubectl apply -f applications/remediator-agent.yaml

STEP 7: VERIFICATION
--------------------

7.1 Check Agent Deployment
kubectl get pods -n nirmata
kubectl get application remediator-agent -n argocd

7.2 Monitor Agent Logs
kubectl logs -n nirmata deployment/remediator-agent --tail=20

7.3 Check for GitHub PRs (after 5+ minutes)
gh pr list --repo YOUR_USERNAME/YOUR_REPO

7.4 Verify Configuration
kubectl get remediator remediator-agent -n nirmata -o yaml
kubectl get secret github-token -n nirmata
kubectl get serviceaccount remediator-agent -n nirmata

EXPECTED RESULTS
----------------

Working Components:
- Agent starts successfully and discovers ArgoCD applications
- Fetches policy violations from application namespace  
- Builds violation context and maps resources to policies
- Successfully authenticates with AWS Bedrock via Pod Identity
- Accesses GitHub API via token secret

Known Issue (RC Version):
- Agent may panic at "Mapping resources to GitHub files" step
- Error: interface conversion: interface {} is nil, not map[string]interface {}
- This is a bug in version 0.2.0-rc4 at argocollector.go:365
- All other components work correctly up to this point

TROUBLESHOOTING
---------------

Common Issues:
1. Pod pending due to resource constraints - scale cluster nodes
2. CRD annotation size limits - use ServerSideApply=true in syncOptions
3. Migration hook failures - disable migration in kyverno values.yaml
4. AWS credentials - verify Pod Identity association is correct
5. GitHub access - verify token permissions include repo scope

Log Analysis:
- "Final selected applications: N" indicates ArgoCD discovery working
- "Fetching policy violations" indicates policy detection working  
- "Building violation context" indicates processing working
- Panic at mapResourcesToFiles indicates RC version bug


APPENDIX A: LLM MODEL SELECTION GUIDE
=====================================

This section explains how to select and configure any LLM model for the remediator agent from scratch.

A.1 Discover Available LLMs
----------------------------
aws bedrock list-foundation-models --region us-west-2 --query 'modelSummaries[*].[modelId,modelName,providerName]' --output table --profile=devtest-sso

This command shows all available foundation models including:
- Anthropic Claude (best for reasoning, security analysis)
- Amazon Nova (AWS native, good performance) 
- Meta Llama (open source, general purpose)
- OpenAI GPT (well known, good performance)
- Mistral (European, privacy focused)

A.2 Check Inference Profiles (High Availability)
------------------------------------------------
aws bedrock list-inference-profiles --region us-west-2 --query 'inferenceProfileSummaries[*].[inferenceProfileId,inferenceProfileName,description]' --output table --profile=devtest-sso

Inference profiles provide:
- Multi-region routing (high availability)
- Automatic failover between regions
- Better performance and reliability
- Recommended over direct foundation models

A.3 Choose Your Model
---------------------
For policy remediation, recommended choices:

Model                           | Use Case                    | Regions | Cost
Claude Sonnet 4                | Best reasoning, security    | 3       | $$$
Claude 3.5 Sonnet v2           | Great performance, lower    | 3       | $$
Nova Premier                   | AWS native, good balance    | 3       | $$
Claude 3.5 Haiku              | Fast, lightweight           | 3       | $

Our Choice: us.anthropic.claude-sonnet-4-20250514-v1:0
- Best reasoning for security policy analysis
- Routes across 3 regions (us-east-1, us-east-2, us-west-2)
- High availability through inference profile

A.4 Get ARNs for Permissions
-----------------------------
aws bedrock list-inference-profiles --region us-west-2 --query 'inferenceProfileSummaries[?inferenceProfileId==`us.anthropic.claude-sonnet-4-20250514-v1:0`]' --profile=devtest-sso

This returns:
1. Inference Profile ARN: Used in IAM policy
   arn:aws:bedrock:us-west-2:844333597536:inference-profile/us.anthropic.claude-sonnet-4-20250514-v1:0

2. Foundation Model ARNs: Used in IAM policy (one per region)
   arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0
   arn:aws:bedrock:us-east-2::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0
   arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0

A.5 Create IAM Policy
---------------------
Create bedrock-policy.json with ALL ARNs from step A.4:

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
                "arn:aws:bedrock:us-west-2:844333597536:inference-profile/us.anthropic.claude-sonnet-4-20250514-v1:0"
            ]
        }
    ]
}

Why multi-region permissions?
- Inference profiles dynamically route between regions
- If primary region is busy, routes to secondary regions  
- You need permissions for ALL regions it might route to

A.6 Configure Remediator Agent
------------------------------
Use the INFERENCE PROFILE ID (not ARN) in custom-values.yaml:

llm:
  enabled: true
  provider: bedrock
  model: "us.anthropic.claude-sonnet-4-20250514-v1:0"  # Inference Profile ID
  region: "us-west-2"  # Primary region for inference profile

Key Understanding:
- Foundation Model ID: anthropic.claude-sonnet-4-20250514-v1:0 (single region)
- Inference Profile ID: us.anthropic.claude-sonnet-4-20250514-v1:0 (multi-region)
- Use Inference Profile ID for high availability

A.7 Apply IAM Policy
--------------------
aws iam create-policy --policy-name RemediatorBedrockAccess --policy-document file://bedrock-policy.json --profile=devtest-sso
aws iam attach-role-policy --role-name remediator-agent-role --policy-arn arn:aws:iam::ACCOUNT:policy/RemediatorBedrockAccess --profile=devtest-sso

This process works for ANY Bedrock model - just replace the model IDs and ARNs accordingly.


