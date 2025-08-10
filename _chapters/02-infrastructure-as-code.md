---
layout: default
title: "Terraform: From Clicks to Code"
---
When I first sketched out this project, I figured I’d spin up my Lambda function, create an S3 bucket, and wire up API Gateway all through the AWS Console. This was obviously less upfront effort, but I quickly realized that approach would bite me as soon as I wanted to scale, collaborate, or even just keep my environments straight. In real-world teams, you never build production features in the same sandbox where you’re experimenting. I wanted to adopt that same discipline: a clean **dev** playground where I could tinker and break things, and a locked-down **prod** environment for anything user-facing. The console alone made this an error-prone nightmare: manual clicks here, copy-and-paste there, and suddenly your “dev” bucket has production data—or vice versa.

### **Enter Terraform**

So I shifted gears and embraced Terraform for everything:

* **One Source of Truth:** Every resource—Lambda, S3, API Gateway, DynamoDB—lives in `.tf` files.
* **Environment Swaps in a Snap:** By parameterizing with workspaces (or different state backends), I can point the same code at a dev account one moment and a prod account the next—just by changing a GitHub Action variable.
* **Git-Driven Confidence:** All changes Github Actions. I get a `terraform plan` preview automatically, so I can peer-review infrastructure like I review code.
* **Secure Secrets Handling:** Access keys, bucket names, and Cognito client IDs live in GitHub Secrets—never hardcoded. Onboarding a new collaborator is as simple as granting them repo-level secrets access, not IAM creds.

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

### CI/CD with GitHub Actions: Click-to-Deploy Simplicity

Beyond just managing infrastructure definitions, I built a GitHub Actions workflow to tie everything together—transforming manual terminal commands into a polished, click-to-deploy experience:

* **Branch-Based Environment Selection:** A workflow input lets you choose `dev` or `prod` when triggering a run. Under the hood, the action sets the `environment` variable accordingly, ensuring the same Terraform code targets the correct AWS account and state bucket.
* **Optional Apply After Plan:** The workflow includes an input flag that controls whether terraform apply runs after terraform plan. This lets you preview the plan output first—checking exactly what will change—before deciding to re-run the workflow with apply enabled. It’s a simple safeguard against unintended infrastructure updates.
* **Secrets & Permissions:** GitHub Secrets store AWS access keys, state bucket names, and Cognito client IDs. The workflow pulls these in securely, eliminating any hardcoded credentials. Role-based access controls in the repo mean only authorized team members can modify the deployment logic.

### GitHub Actions Workflow (`.github/workflows/terraform.yml`)

```yaml
name: "Terraform Deploy"

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Which environment to target (dev or prod)"
        required: true
        type: choice
        default: "dev"
        options:
          - dev
          - prod
      confirm_apply:
        description: "Set to true to run the apply step (default=false → plan only)"
        required: false
        type: boolean
        default: false

jobs:
  terraform:
    runs-on: ubuntu-latest

    env:
      AWS_REGION: ap-southeast-2

      # Dev vs Prod credentials
      AWS_ACCESS_KEY_ID_DEV:     ${{ secrets.AWS_ACCESS_KEY_ID_DEV }}
      AWS_SECRET_ACCESS_KEY_DEV: ${{ secrets.AWS_SECRET_ACCESS_KEY_DEV }}
      AWS_ACCESS_KEY_ID_PROD:    ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
      AWS_SECRET_ACCESS_KEY_PROD: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
      AWS_UPLOAD_API_SECRET_DEV: ${{ secrets.UPLOAD_API_SECRET_DEV }}
      AWS_UPLOAD_API_SECRET_PROD: ${{ secrets.UPLOAD_API_SECRET_PROD }}
      AWS_UPLOAD_API_URL_DEV:    ${{ secrets.AWS_API_URL_DEV }}
      AWS_UPLOAD_API_URL_PROD:   ${{ secrets.AWS_API_URL_PROD }}
      AWS_LIST_API_URL_REDIRECT_DEV: ${{ secrets.AWS_API_URL_LIST_FILES_DEV}}

      # Backend resources
      STATE_BUCKET_DEV:   ${{ secrets.STATE_BUCKET_DEV }}
      STATE_BUCKET_PROD:  ${{ secrets.STATE_BUCKET_PROD }}
      DYNAMODB_TABLE:     ${{ secrets.DYNAMODB_TABLE }}
      STATE_PREFIX:       ${{ secrets.STATE_PREFIX }}

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Select AWS credentials and bucket
        run: |
          if [[ "${{ github.event.inputs.environment }}" == "dev" ]]; then
            echo "AWS_ACCESS_KEY_ID=${{ env.AWS_ACCESS_KEY_ID_DEV }}" >> $GITHUB_ENV
            echo "AWS_SECRET_ACCESS_KEY=${{ env.AWS_SECRET_ACCESS_KEY_DEV }}" >> $GITHUB_ENV
            echo "STATE_BUCKET=${{ env.STATE_BUCKET_DEV }}" >> $GITHUB_ENV
          else
            echo "AWS_ACCESS_KEY_ID=${{ env.AWS_ACCESS_KEY_ID_PROD }}" >> $GITHUB_ENV
            echo "AWS_SECRET_ACCESS_KEY=${{ env.AWS_SECRET_ACCESS_KEY_PROD }}" >> $GITHUB_ENV
            echo "STATE_BUCKET=${{ env.STATE_BUCKET_PROD }}" >> $GITHUB_ENV
          fi

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: "1.5.0"

      - name: Determine target
        id: env
        run: |
          echo "WORK_DIR=live/${{ github.event.inputs.environment }}" >> $GITHUB_OUTPUT

      - name: Terraform Init
        working-directory: ${{ steps.env.outputs.WORK_DIR }}
        run: |
          terraform init -input=false \
            -backend-config="bucket=${{ env.STATE_BUCKET }}" \
            -backend-config="key=${{ env.STATE_PREFIX }}/${{ github.event.inputs.environment }}/terraform.tfstate" \
            -backend-config="region=${{ env.AWS_REGION }}" \
            -backend-config="dynamodb_table=${{ env.DYNAMODB_TABLE }}"

      - name: Terraform Validate
        working-directory: ${{ steps.env.outputs.WORK_DIR }}
        run: terraform validate

      - name: Terraform Plan
        working-directory: ${{ steps.env.outputs.WORK_DIR }}
        run: |
          if [[ "${{ github.event.inputs.environment }}" == "dev" ]]; then
            SECRET=${{ env.AWS_UPLOAD_API_SECRET_DEV }}
            API_URL=${{ env.AWS_UPLOAD_API_URL_DEV }}
            COGNITO_CLIENT_ID=${{ secrets.COGNITO_CLIENT_ID_DEV }}
            COGNITO_DOMAIN=${{ secrets.COGNITO_DOMAIN_DEV }}
            LOGIN_URL=${{ secrets.LOGIN_URL_DEV }}
            REDIRECT_URI_LIST=${{ secrets.AWS_API_LIST_REDIRECT_URL_DEV }}
            API_URL_LIST_FILES=${{ secrets.AWS_API_URL_LIST_FILES_DEV }}
          else
            SECRET=${{ env.AWS_UPLOAD_API_SECRET_PROD }}
            API_URL=${{ env.AWS_UPLOAD_API_URL_PROD }}
            API_URL_LIST_FILES=${{ secrets.AWS_API_URL_LIST_FILES_PROD }}
            REDIRECT_URI_LIST=${{ secrets.AWS_API_URL_LIST_FILES_PROD }}
            COGNITO_CLIENT_ID=${{ secrets.COGNITO_CLIENT_ID_PROD }}
            COGNITO_DOMAIN=${{ secrets.COGNITO_DOMAIN_PROD }}
            LOGIN_URL=${{ secrets.LOGIN_URL_PROD }}
          fi

          terraform plan \
            -var="aws_region=${{ env.AWS_REGION }}" \
            -var="state_bucket=${{ env.STATE_BUCKET }}" \
            -var="state_prefix=${{ env.STATE_PREFIX }}" \
            -var="dynamodb_table=${{ env.DYNAMODB_TABLE }}" \
            -var="upload_api_secret=$SECRET" \
            -var="upload_api_url=$API_URL" \
            -var="login_redirect_url=$LOGIN_URL" \
            -var="logout_redirect_url=$LOGIN_URL" \
            -var="cognito_client_id=$COGNITO_CLIENT_ID" \
            -var="cognito_domain=$COGNITO_DOMAIN" \
            -var="redirect_uri_list=$AWS_LIST_API_URL_REDIRECT_DEV" \
            -var="list_api_url=$API_URL_LIST_FILES" \
            -out=tfplan

      - name: Terraform Apply (manual)
        if: ${{ github.event.inputs.confirm_apply == 'true' }}
        working-directory: ${{ steps.env.outputs.WORK_DIR }}
        run: terraform apply -auto-approve tfplan
        
      - name: Invalidate CloudFront (index.html)
        if: ${{ github.event.inputs.confirm_apply == 'true' }}
        run: |
          if [[ "${{ github.event.inputs.environment }}" == "dev" ]]; then
            DIST_ID="${{ secrets.CF_DIST_ID_DEV }}"
          else
            DIST_ID="${{ secrets.CF_DIST_ID_PROD }}"
          fi
          aws cloudfront create-invalidation --distribution-id "$DIST_ID" --paths "/index.html" "/"


```
This IaC + CI/CD foundation set the stage for every other layer of the project—authentication flows, event-driven logging, CORS strategies, and more. With it in place, I could focus on building features, confident that my infrastructure would remain consistent, reproducible, and secure.

### Key points
- **Single source of truth:** Every AWS resource (S3, CloudFront, API Gateway, Lambda, DynamoDB, SES) is declared in `.tf`—no console drift.
- **Dev/Prod parity:** Same code, different **backend state** and variables per environment; the workflow selects the target env at run time.
- **Safe by default:** CI runs `terraform validate` + `plan`; `apply` only happens when you explicitly toggle `confirm_apply=true`.
- **Locked, versioned state:** S3 + DynamoDB state locking prevents concurrent applies and makes rollbacks/audits straightforward.
- **Secrets stay secret:** AWS creds, IDs, and OAuth config live in **GitHub Secrets**—never hardcoded in repo or templates.
- **Deterministic deploys:** Module inputs (region, stage, ids) come from the workflow; outputs (e.g., CloudFront URL) are emitted by Terraform for later steps.

-----------------------------

