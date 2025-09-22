---
layout: default
title: "GitHub OIDC: From Static Keys to Short-Lived Roles"
---

When I kicked off Project 2, I decided to improve the security of my CI/CD by removing the dependency on my static AWS keys. I wanted **Terraform in charge** like Project 1—but with CI assuming **short-lived credentials via GitHub OIDC**. The result is cleaner security, tighter environment isolation, and a smoother deploy loop.

### **Enter OIDC (keep Terraform in charge)**

So I kept `.tf` as the single source of truth and wired CI to assume roles at runtime:

- **Ephemeral credentials only:** GitHub → OIDC → STS; nothing to rotate or leak.
- **Per-env roles:** one role for `dev`, one for `prod`, same workflow.
- **Guardrails first:** explicit **deny outside region** and **deny TF state deletes**.

### Minimal IAM trust policy (repo-scoped, any branch)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": ["sts:AssumeRoleWithWebIdentity", "sts:TagSession"],
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:Roshan-AWS-Personal/roshan-aws-personal.github.io:ref:refs/heads/*"
      }
    }
  }]
}

```
### Trust policy tied to one repo (any branch)
```hcl
data "aws_iam_policy_document" "gha_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity", "sts:TagSession"]
    principals { type = "Federated", identifiers = [data.aws_iam_openid_connect_provider.github.arn] }
    condition {
      test = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values = ["sts.amazonaws.com"]
    }
    condition {
      test = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values = ["repo:${var.github_owner}/${var.github_repo}:ref:refs/heads/*"]
    }
  }
}

resource "aws_iam_role" "gha" {
  name               = "${var.project}-${var.env}-gha-role"  # e.g., ai-kb-dev-gha-role
  assume_role_policy = data.aws_iam_policy_document.gha_trust.json
  max_session_duration = 3600
  tags = { Project = var.project, Env = var.env, ManagedBy = "terraform" }
}

# Guardrail: deny outside chosen region (exclude global services)
data "aws_iam_policy_document" "deny_outside_region" {
  statement {
    sid        = "DenyOutsideRegion"
    effect     = "Deny"
    not_actions = ["iam:*","sts:*","cloudfront:*","route53:*","waf:*","wafv2:*","shield:*",
                   "organizations:*","support:*","budgets:*","globalaccelerator:*"]
    resources = ["*"]
    condition {
      test = "StringNotEquals"
      variable = "aws:RequestedRegion"
      values = [var.region]  # ap-southeast-2
    }
  }
}
resource "aws_iam_policy" "deny_outside_region" {
  name   = "${var.project}-${var.env}-deny-outside-region"
  policy = data.aws_iam_policy_document.deny_outside_region.json
}

# Guardrail: deny deleting TF state & lock table
data "aws_iam_policy_document" "deny_tf_state_deletes" {
  statement {
    sid     = "DenyDeleteTfState"
    effect  = "Deny"
    actions = ["s3:DeleteBucket","s3:DeleteObject","s3:PutBucketVersioning","dynamodb:DeleteTable"]
    resources = [
      "arn:aws:s3:::${var.state_bucket}",
      "arn:aws:s3:::${var.state_bucket}/*",
      "arn:aws:dynamodb:${var.region}:${data.aws_caller_identity.me.account_id}:table/${var.dynamodb_table}"
    ]
  }
}
resource "aws_iam_policy" "deny_tf_state_deletes" {
  name   = "${var.project}-${var.env}-deny-tf-state-deletes"
  policy = data.aws_iam_policy_document.deny_tf_state_deletes.json
}

# Temporary while building (replace with least-privilege later)
resource "aws_iam_role_policy_attachment" "admin" {
  role = aws_iam_role.gha.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}
resource "aws_iam_role_policy_attachment" "guard_region" {
  role = aws_iam_role.gha.name
  policy_arn = aws_iam_policy.deny_outside_region.arn
}
resource "aws_iam_role_policy_attachment" "guard_state" {
  role = aws_iam_role.gha.name
  policy_arn = aws_iam_policy.deny_tf_state_deletes.arn
}
```

```yaml
name: "Terraform Deploy"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "dev or prod"
        type: choice
        options: [dev, prod]
        default: dev
      confirm_apply:
        description: "run apply after plan?"
        type: boolean
        default: false

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    env:
      AWS_REGION: ap-southeast-2
      # roles + state per env (stored as GitHub Secrets)
      ROLE_ARN_DEV:  ${{ secrets.ROLE_ARN_DEV }}
      ROLE_ARN_PROD: ${{ secrets.ROLE_ARN_PROD }}
      STATE_BUCKET_DEV:  ${{ secrets.STATE_BUCKET_DEV }}
      STATE_BUCKET_PROD: ${{ secrets.STATE_BUCKET_PROD }}
      STATE_PREFIX:      ${{ secrets.STATE_PREFIX }}
      DYNAMODB_TABLE:    ${{ secrets.DYNAMODB_TABLE }}
      CHAT_API_DOMAIN_DEV:  ${{ secrets.CHAT_API_DOMAIN }}
      CHAT_API_DOMAIN_PROD: ${{ secrets.CHAT_API_DOMAIN_PROD }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Select env
        id: sel
        run: |
          if [[ "${{ github.event.inputs.environment }}" == "dev" ]]; then
            echo "ROLE_TO_ASSUME=${{ env.ROLE_ARN_DEV }}"   >> $GITHUB_ENV
            echo "STATE_BUCKET=${{ env.STATE_BUCKET_DEV }}" >> $GITHUB_ENV
            echo "CHAT_DOMAIN=${{ env.CHAT_API_DOMAIN_DEV }}" >> $GITHUB_ENV
          else
            echo "ROLE_TO_ASSUME=${{ env.ROLE_ARN_PROD }}"  >> $GITHUB_ENV
            echo "STATE_BUCKET=${{ env.STATE_BUCKET_PROD }}" >> $GITHUB_ENV
            echo "CHAT_DOMAIN=${{ env.CHAT_API_DOMAIN_PROD }}" >> $GITHUB_ENV
          fi
          echo "WORK_DIR=live/${{ github.event.inputs.environment }}" >> $GITHUB_OUTPUT

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: tf-${{ github.run_id }}

      - name: Who am I?
        run: aws sts get-caller-identity

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: "1.5.0"

      - name: Terraform Init (remote state)
        working-directory: ${{ steps.sel.outputs.WORK_DIR }}
        run: |
          terraform init -input=false \
            -backend-config="bucket=${{ env.STATE_BUCKET }}" \
            -backend-config="key=${{ env.STATE_PREFIX }}/${{ github.event.inputs.environment }}/terraform.tfstate" \
            -backend-config="region=${{ env.AWS_REGION }}" \
            -backend-config="dynamodb_table=${{ env.DYNAMODB_TABLE }}"

      - name: Terraform Plan
        working-directory: ${{ steps.sel.outputs.WORK_DIR }}
        run: |
          terraform plan \
            -var="aws_region=${{ env.AWS_REGION }}" \
            -var="chat_api_domain=${{ env.CHAT_DOMAIN }}" \
            -out=tfplan

      - name: Terraform Apply (manual)
        if: ${{ github.event.inputs.confirm_apply == 'true' }}
        working-directory: ${{ steps.sel.outputs.WORK_DIR }}
        run: terraform apply -auto-approve tfplan
```

## One-time OIDC bootstrap (what I did at the terminal)
To break the chicken-and-egg (the role must exist before CI can assume it), I created the OIDC role **once** with local AWS creds, grabbed its ARN, and fed that into GitHub as a secret.

**1) Add an output to `oidc.tf` so Terraform prints the role ARN**
```hcl
output "gha_role_arn" {
  description = "IAM role ARN to be assumed by GitHub Actions (dev)"
  value       = aws_iam_role.gha.arn
}
```

**2) Add an output to `oidc.tf` so Terraform prints the role ARN**
```bash
cd live/dev

# init with your remote state backend
terraform init \
  -backend-config="bucket=<STATE_BUCKET_DEV>" \
  -backend-config="key=<STATE_PREFIX>/dev/terraform.tfstate" \
  -backend-config="region=ap-southeast-2" \
  -backend-config="dynamodb_table=<DYNAMODB_TABLE>"

# apply (you can target only the OIDC resources to be surgical)
terraform apply -auto-approve \
  -var='github_owner=<YOUR_GITHUB_OWNER>' \
  -var='github_repo=<YOUR_REPO_NAME>' \
  -var='allowed_branches=["main","dev"]' \
  -var='state_bucket=<STATE_BUCKET_DEV>' \
  -var='state_prefix=<STATE_PREFIX>' \
  -var='dynamodb_table=<DYNAMODB_TABLE>' \
  -var='region=ap-southeast-2' \
  -var='env=dev'
# optional, to limit changes:
# -target=aws_iam_role.gha \
# -target=aws_iam_policy.deny_outside_region \
# -target=aws_iam_policy.deny_tf_state_deletes
```
**3) Capture the ARN and save it as a GitHub secret**
```bash
terraform output -raw gha_role_arn
# copy the value → GitHub > Settings > Secrets and variables > Actions > New secret
```

**4) Capture the ARN and save it as a GitHub secret**
Add a quick echo before configure-aws-credentials:
```yaml
- name: Echo selected role
  run: echo "ROLE_TO_ASSUME=${{ env.ROLE_TO_ASSUME }}"```

---

