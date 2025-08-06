---
layout: default
title: 
---
-----------------------------
## Infrastructure as Code: From Clicks to Code
When I first sketched out this project, I figured I’d spin up my Lambda function, create an S3 bucket, and wire up API Gateway all through the AWS Console. Sure, it meant less upfront effort, but I quickly realized that approach would bite me as soon as I wanted to scale, collaborate, or even just keep my environments straight.

**“Dev” vs. “Prod”: Separate your Playgrounds**

In real-world teams, you never build production features in the same sandbox where you’re experimenting. I wanted to adopt that same discipline: a clean **dev** playground where I could tinker and break things, and a locked-down **prod** environment for anything user-facing. The console alone made this an error-prone nightmare: manual clicks here, copy-and-paste there, and suddenly your “dev” bucket has production data—or vice versa.

### **Enter Terraform**

So I shifted gears and embraced Terraform for everything:

* **One Source of Truth:** Every resource—Lambda, S3, API Gateway, DynamoDB—lives in `.tf` files. No secret console steps hiding in my brain.
* **Environment Swaps in a Snap:** By parameterizing with workspaces (or different state backends), I can point the same code at a dev account one moment and a prod account the next—just by changing a GitHub Action variable.
* **Git-Driven Confidence:** All changes go through pull requests. I get a `terraform plan` preview automatically, so I can peer-review infrastructure like I review code.
* **Secure Secrets Handling:** Access keys, bucket names, and Cognito client IDs live in GitHub Secrets—never hardcoded. Onboarding a new collaborator is as simple as granting them repo-level secrets access, not IAM creds.
* **Repeatability & Disaster Recovery:** Destroying and re-creating an environment is as easy as 
```bash
terraform destroy && terraform apply
```


No more accidental cloud sprawl or “how-did-that-thing-get-created?” mysteries.

### Sample Terraform Code Snippet

```hcl
module "upload_api" {
  source        = "./modules/api-gateway"
  environment   = var.environment
  lambda_source = "../lambdas/upload"
  bucket_name   = "${var.project_name}-${var.environment}-uploads"
}

variable "environment" {
  type    = string
  default = "dev"
}
```

## **CI/CD with GitHub Actions: Click-to-Deploy Simplicity**

Beyond just managing infrastructure definitions, I built a GitHub Actions workflow to tie everything together—transforming manual terminal commands into a polished, click-to-deploy experience:

* **Branch-Based Environment Selection:** A workflow input lets you choose `dev` or `prod` when triggering a run. Under the hood, the action sets the `environment` variable accordingly, ensuring the same Terraform code targets the correct AWS account and state bucket.
* **Manual Approvals & Protected Branches:** By protecting the `main` branch and requiring at least one approver before merging, I ensure `prod` deployments never happen by accident. Once merged, the `apply` step runs automatically, promoting changes to production with confidence.
* **Plan Previews in PR Checks:** Every pull request against either branch triggers a `terraform plan` step. The resulting plan summary is posted as a check comment—so reviewers see exactly what infrastructure changes will occur.
* **Secrets & Permissions:** GitHub Secrets store AWS access keys, state bucket names, and Cognito client IDs. The workflow pulls these in securely, eliminating any hardcoded credentials. Role-based access controls in the repo mean only authorized team members can modify the deployment logic.

### GitHub Actions Workflow (`.github/workflows/terraform.yml`)

```yaml
name: "Terraform Infrastructure"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Choose environment.'
        required: true
        default: dev
        type: choice
        options:
          - dev
          - prod
  pull_request:
    branches:
      - dev
      - main

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-2

      - name: Terraform Init
        run: terraform init \
          -backend-config="bucket=my-tf-state-${{ github.event.inputs.environment }}"

      - name: Terraform Plan
        run: terraform plan \
          -var="env=${{ github.event.inputs.environment }}"

      - name: Terraform Apply (Prod Only)
        if: github.event.inputs.environment == 'prod' && github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve \
          -var="env=prod"
```

## **Why This Matters**

* **Scalability:** As soon as I added Cognito, CloudFront, or SES, I simply dropped in new Terraform modules—no manual console gymnastics.
* **Collaboration:** My teammates (or future clients) can spin up a full stack in their own AWS accounts.
* **Auditability:** I can trace exactly *when* and *why* a resource changed by looking at the Terraform state and Git history.

This IaC + CI/CD foundation set the stage for every other layer of the project—authentication flows, event-driven logging, CORS strategies, and more. With it in place, I could focus on building features, confident that my infrastructure would remain consistent, reproducible, and secure.



