# AWS Serverless CI/CD Guide for Frappe Docker

This guide explains how to replace the Jenkins CI workflow with a fully serverless AWS Native solution using **AWS CodeBuild** and **AWS CodePipeline**.

## Architecture Comparison

| Feature | Old (Jenkins) | New (AWS Serverless) |
|:--- |:--- |:--- |
| **Orchestrator** | Jenkins Master (Docker Container) | **AWS CodePipeline** |
| **Build Agent** | Jenkins Agent (Docker Sibling) | **AWS CodeBuild** (Ephemeral, Scaling) |
| **Secrets** | Jenkins Credentials Plugin | **AWS Secrets Manager / Parameter Store** |
| **Trigger** | Webhooks to Jenkins URL | **GitHub Webhook / CodeStar Connection** |
| **Cost** | Always-on EC2/Container | **Pay-per-minute** of build time only |

---

## 1. Preparation

### 1.1 Prerequisites
- AWS Account with `ap-south-1` (Mumbai) region active.
- Existing ECR Repository: `erp-ventures`.
- GitHub Repository with `apps.json` and `Dockerfile`.
- `buildspec.yml` added to your repository root (see Step 2).

### 1.2 Store GitHub Configs
Since we are moving away from Jenkins, we need a secure place to store the GitHub Token used for cloning your private apps (Shopbridge, etc) inside the Docker build.

1.  Go to **AWS Systems Manager** > **Parameter Store**.
2.  Click **Create parameter**.
    *   **Name**: `/frappe-docker/github-token`
    *   **Type**: `SecureString`
    *   **Value**: Your GitHub Personal Access Token (PAT).
3.  Click **Create parameter**.

---

## 2. Commit `buildspec.yml`

Ensure the `buildspec.yml` file is present in the root of your repository. This file tells AWS CodeBuild what to do (similar to `Jenkinsfile`).

**Key sections in `buildspec.yml`:**
- **Secrets Manager/Parameter Store**: Fetches `/frappe-docker/github-token` securely.
- **Pre-build**: Logins to ECR, encodes `apps.json` with the token.
- **Build**: Runs `docker build`.
- **Post-build**: Pushes images to ECR.

---

## 3. Create CodeBuild Project

1.  Go to **AWS CodeBuild Console** > **Create build project**.
2.  **Project configuration**:
    *   **Project name**: `frappe-docker-build`
3.  **Source**:
    *   **Source provider**: `GitHub`.
    *   **Repository**: Connect your account and select `casavaventures/frappe_docker`.
    *   **Source version**: `main` (or your preferred branch).
    *   **Webhook**: Check "Rebuild every time a code change is pushed to this repository" if you want auto-builds without CodePipeline. (Recommend leaving unchecked if using CodePipeline).
4.  **Environment**:
    *   **Environment image**: `Managed image`.
    *   **Operating system**: `Ubuntu`.
    *   **Runtime**: `Standard`.
    *   **Image**: `aws/codebuild/standard:7.0` (or latest).
    *   **Privileged**: **CHECK THIS BOX** (Crucial for Docker build).
    *   **Service role**: Create a new service role (e.g., `codebuild-frappe-service-role`).
5.  **Buildspec**:
    *   Select "Use a buildspec file".
6.  **Artifacts**:
    *   Type: `No artifacts` (We are pushing to ECR, not saving files).
7.  **Logs**:
    *   CloudWatch logs: Checked.

### 3.1 Configure Environment Variables
In the CodeBuild project settings (Edit > Environment), add these **Plaintext** variables:

| Name | Value |
|:--- |:--- |
| `AWS_DEFAULT_REGION` | `ap-south-1` |
| `AWS_ACCOUNT_ID` | `587082268194` |
| `ECR_REPO` | `erp-ventures` |

### 3.2 Add Permissions to Role
The CodeBuild Service Role created above needs permission to ECR and Parameter Store.

1.  Go to **IAM Console** > **Roles**.
2.  Search for `codebuild-frappe-service-role`.
3.  Add Permissions > **Attach policies**:
    *   `AmazonEC2ContainerRegistryPowerUser` (Allows Push/Pull).
    *   `AmazonSSMReadOnlyAccess` (Allows reading Parameter Store).

---

## 4. (Optional) Create Pipeline

To create a full Continuous Delivery workflow:

1.  Go to **AWS CodePipeline** > **Create pipeline**.
2.  **Name**: `frappe-docker-pipeline`.
3.  **Source Stage**:
    *   **Source provider**: `GitHub (Version 2)`.
    *   **Connection**: Create a new connection to GitHub (popup auth).
    *   **Repo**: `casavaventures/frappe_docker`.
    *   **Branch**: `main`.
4.  **Build Stage**:
    *   **Build provider**: `AWS CodeBuild`.
    *   **Project name**: `frappe-docker-build` (Created in Step 3).
5.  **Deploy Stage**:
    *   Skip for now (We are deploying via Docker Image update, not CodeDeploy).

---

## 5. Usage

### Triggering a Build
- **Manual**: Go to CodeBuild > `frappe-docker-build` > **Start build**.
- **Automatic**: Push a commit to `main` (if Webhook was enabled in Step 3 or if using CodePipeline).

### Verification
1.  Watch the **Build logs** in CodeBuild console.
2.  Verify steps: `Login` -> `Apps.json encoding` -> `Docker Build` -> `Docker Push`.
3.  Go to **ECR Console** and verify a new image tagged `latest` and `1.0.<build-id>` exists.

---

## Troubleshooting

- **Error: `docker: command not found`**: Ensure you selected a **Standard** image (not Base) in CodeBuild environment.
- **Error: `Cannot connect to the Docker daemon`**: Ensure **Privileged** flag is checked in CodeBuild environment settings.
- **Error: `AccessDeniedException` for Parameter Store**: Ensure the CodeBuild IAM Role has `ssm:GetParameter` permission.
- **Error: `AccessDeniedException` for ECR**: Ensure the CodeBuild IAM Role has `ecr:GetAuthorizationToken` and `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, etc.
