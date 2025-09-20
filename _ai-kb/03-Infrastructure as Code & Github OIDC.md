---
layout: default
title: "Infrastructure as Code & GitHub OIDC"
---

I kept the same discipline from Project 1—**Terraform for shape, CI for motion**—but this time I set up **GitHub OIDC** cleanly from the start so deploys don’t rely on long-lived AWS keys.

## Why OIDC here
- **No static keys** in repo or runners; short-lived, auto-rotated credentials.
- **Per-env roles** (dev/prod) give least-privilege and clearer blast radius.
- Works for both **Terraform plans/applies** and **Lambda code-only deploys**.

---

## AWS: the trust relationship (per environment)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:Roshan-AWS-Personal/roshan-aws-personal.github.io:*"
      }
    }
  }]
}

```yaml
name: "Terraform Deploy"

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, prod]
        default: dev
      confirm_apply:
        type: boolean
        default: false

jobs:
  tf:
    runs-on: ubuntu-latest
    permissions:
      id-token: write     # OIDC!
      contents: read

    env:
      AWS_REGION: ap-southeast-2
      CHAT_API_DOMAIN: ${{ vars.CHAT_API_DOMAIN }} # org/repo Variable, not secret

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS creds via OIDC
        uses: aws-actions/configure-aws-credentials@v3
        with:
          role-to-assume: ${{ github.event.inputs.environment == 'prod'
            && 'arn:aws:iam::<PROD_ACCT>:role/github-oidc-deploy'
            || 'arn:aws:iam::<DEV_ACCT>:role/github-oidc-deploy' }}
          aws-region: ${{ env.AWS_REGION }}

      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.6.6 }

      - name: Terraform Init (remote state)
        working-directory: live/${{ github.event.inputs.environment }}
        run: |
          terraform init -input=false \
            -backend-config="bucket=${{ secrets.STATE_BUCKET }}" \
            -backend-config="key=ai-kb/${{ github.event.inputs.environment }}/tfstate" \
            -backend-config="region=${{ env.AWS_REGION }}" \
            -backend-config="dynamodb_table=${{ secrets.DDB_LOCK_TABLE }}"

      - name: Terraform Plan
        working-directory: live/${{ github.event.inputs.environment }}
        run: |
          terraform plan \
            -var="aws_region=${{ env.AWS_REGION }}" \
            -var="chat_api_domain=${{ env.CHAT_API_DOMAIN }}" \
            -out=tfplan

      - name: Terraform Apply (manual gate)
        if: ${{ github.event.inputs.confirm_apply == true }}
        working-directory: live/${{ github.event.inputs.environment }}
        run: terraform apply -auto-approve tfplan
```
Notes

permissions.id-token: write is required for OIDC.

We pass chat_api_domain as a var so HTML/JS never hard-codes it.

State is namespaced under ai-kb/<env>/tfstate.


```yaml
name: "Deploy Lambda Code"

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, prod]
        default: dev

jobs:
  code:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read }

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v3
        with:
          role-to-assume: ${{ github.event.inputs.environment == 'prod'
            && 'arn:aws:iam::<PROD_ACCT>:role/github-oidc-deploy'
            || 'arn:aws:iam::<DEV_ACCT>:role/github-oidc-deploy' }}
          aws-region: ap-southeast-2

      - name: Zip and upload Query Lambda
        run: |
          cd lambda/query && zip -r ../query.zip .
          aws lambda update-function-code \
            --function-name ai-kb-${{ github.event.inputs.environment }}-query \
            --zip-file fileb://../query.zip
```

```hcl
# variables.tf
variable "chat_api_domain" { type = string }
variable "aws_region"      { type = string }

# locals.tf
locals {
  chat_api_domain = var.chat_api_domain
}

# example usage
output "chat_api_domain" { value = local.chat_api_domain }
```

Pitfalls I hit (and fixed)

Missing id-token permission → GitHub couldn’t mint a token; auth failed.

Trust policy too strict → wrong sub pattern; I switched to repo:ORG/REPO:* while iterating, then tightened.

Wrong role per env → I now pick role by input (dev/prod) to avoid cross-account surprises.

Leaking config into HTML → moved CHAT_API_DOMAIN to a workflow variable and pass as a Terraform var.

Result: fully reproducible deploys, no long-lived credentials, and fast code-only releases while infrastructure stays locked behind Terraform.