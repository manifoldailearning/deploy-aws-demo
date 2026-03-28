
1. **AWS account** with a payment method (required for most services).
2. **Email and phone** verified on the account.
3. **GitHub repository** containing this project (your fork or copy).
4. **Tools on your laptop** (for first-time cluster and ECR setup):
   - [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
   - `kubectl` ([install guide](https://kubernetes.io/docs/tasks/tools/))
   - [eksctl](https://eksctl.io/installation/) (simplest way to create EKS)
   - Docker (optional locally; CI builds in GitHub)

## 1. Create an ECR repository

1. In the console: **Elastic Container Registry** → **Repositories** → **Create repository**.
2. Name it something stable (for example `support-ops-agent`). **Private** is fine.
3. Note the **repository name**; you will use it as the GitHub secret `ECR_REPOSITORY`.

## 2. Create an EKS cluster (minimal path with eksctl)

Example: one small managed node group suitable for learning (not HA; adjust for production).

```bash
export AWS_REGION=us-east-1   # your region
export CLUSTER_NAME=support-ops-demo

eksctl create cluster \
  --name support-ops-demo  \
  --region us-east-1 \
  --nodes 2 \
  --node-type t3.small \
  --managed
```

3. IAM user for GitHub Actions (push ECR + kubectl)

The workflow uses **long-lived access keys** (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`). Prefer **OIDC + short-lived roles** in real production; keys are simpler for a first setup.

4. GitHub repository secrets

In GitHub: **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

| Secret name | Value |
|-------------|--------|
| `AWS_ACCESS_KEY_ID` | IAM user key for CI. |
| `AWS_SECRET_ACCESS_KEY` | Matching secret key. |
| `AWS_REGION` | e.g. `us-east-1`. |
| `AWS_ACCOUNT_ID` | 12-digit account ID (`aws sts get-caller-identity --query Account --output text`). |
| `ECR_REPOSITORY` | ECR repo name (not the full URI). |
| `EKS_CLUSTER_NAME` | Your cluster name. |
| `K8S_NAMESPACE` | Must match manifests: `support-ops-agent`. |

The workflow `.github/workflows/deploy.yml` reads these names exactly.

### 4 Enable IAM roles for service accounts (IRSA)

Your pods need AWS permissions for **Secrets Manager** and (if enabled) **CloudWatch Logs**. EKS uses **IRSA**.

Create the OIDC provider if it does not exist (eksctl often does this automatically; if not):

```bash
eksctl utils associate-iam-oidc-provider --cluster "support-ops-demo" --region "us-east-1" --approve
```

###  5 Create the Policy:
``bash
aws iam create-policy \
  --policy-name support-ops-agent-irsa-policy \
  --policy-document file://irsa-support-ops-agent-policy.json

```

###  5 Create the Policy:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

Get OIDC issuer :
```bash
aws eks describe-cluster \
  --name support-ops-demo \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text | sed 's#^https://##'

```

Get he IODC Provider and maintain the draft_irsa-trust-policy copy.json 

Create IAM Role:
```bash
aws iam create-role \
  --role-name support-ops-agent-irsa-role \
  --assume-role-policy-document file://irsa-trust-policy.json
```

```bash
aws iam attach-role-policy \
  --role-name support-ops-agent-irsa-role \
  --policy-arn arn:aws:iam::074388197187:policy/support-ops-agent-irsa-policy
```

Get the Role ARN:
```
aws iam get-role \
  --role-name support-ops-agent-irsa-role \
  --query "Role.Arn" \
  --output text
```

ARN Output --> arn:aws:iam::074388197187:role/support-ops-agent-irsa-role

### Annotate the Kubernetes ServiceAccount
Edit deployment/k8s/serviceaccount.yaml , and update the Account id & ARN there,

Run the following command:
kubectl apply -f deployment/k8s/serviceaccount.yaml