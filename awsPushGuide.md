# Quick Push Guide - Your AWS ECR

## Your AWS Details
- **AWS Account ID**: 587082268194
- **AWS Region**: ap-south-1 (Mumbai)
- **ECR Registry**: 587082268194.dkr.ecr.ap-south-1.amazonaws.com

---

## Step-by-Step Commands

### Step 1: Create ECR Repository (One Time Only)

**Git Bash:**
```bash
aws ecr create-repository \
  --repository-name erp-ventures \
  --region ap-south-1
```

**PowerShell:**
```powershell
aws ecr create-repository `
  --repository-name erp-ventures `
  --region ap-south-1
```

---

### Step 2: Authenticate Docker to ECR

**Git Bash:**
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 587082268194.dkr.ecr.ap-south-1.amazonaws.com
```

**PowerShell:**
```powershell
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 587082268194.dkr.ecr.ap-south-1.amazonaws.com
```

Expected output: `Login Succeeded`

---

### Step 3: Tag Your Image

**Git Bash:**
```bash
# Get your current version
export VERSION=$(cat version.txt)

# Tag with version
docker tag erp-ventures:$VERSION 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:$VERSION

# Tag as latest
docker tag erp-ventures:latest 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest
```

**PowerShell:**
```powershell
# Get your current version
$VERSION = Get-Content version.txt

# Tag with version
docker tag erp-ventures:$VERSION 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:$VERSION

# Tag as latest
docker tag erp-ventures:latest 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest
```

**Or manually (replace 1.0.0 with your version):**
```bash
docker tag erp-ventures:1.0.0 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:1.0.0
docker tag erp-ventures:latest 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest
```

---

### Step 4: Push to ECR

**Git Bash:**
```bash
# Push version tag
docker push 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:$VERSION

# Push latest tag
docker push 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest
```

**PowerShell:**
```powershell
# Push version tag
docker push 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:$VERSION

# Push latest tag
docker push 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest
```

**Or manually:**
```bash
docker push 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:1.0.0
docker push 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest
```

---

### Step 5: Verify Push

```bash
aws ecr describe-images --repository-name erp-ventures --region ap-south-1
```

---

## Complete Automated Script

### Git Bash Script (push-to-ecr.sh)

```bash
#!/bin/bash

set -e

echo "=== Push to AWS ECR (Mumbai Region) ==="

# Configuration
AWS_REGION="ap-south-1"
AWS_ACCOUNT_ID="587082268194"
REPOSITORY_NAME="erp-ventures"
ECR_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPOSITORY_NAME"

# Get version
if [ -f version.txt ]; then
    VERSION=$(cat version.txt)
else
    VERSION="latest"
fi

echo "AWS Account ID: $AWS_ACCOUNT_ID"
echo "AWS Region: $AWS_REGION"
echo "ECR URI: $ECR_URI"
echo "Version: $VERSION"

# Step 1: Authenticate Docker to ECR
echo "Authenticating to ECR..."
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Step 2: Tag image
echo "Tagging image..."
docker tag erp-ventures:$VERSION $ECR_URI:$VERSION
docker tag erp-ventures:latest $ECR_URI:latest

# Step 3: Push image
echo "Pushing image with version $VERSION..."
docker push $ECR_URI:$VERSION

echo "Pushing image with tag latest..."
docker push $ECR_URI:latest

# Step 4: Verify
echo "Verifying push..."
aws ecr describe-images --repository-name $REPOSITORY_NAME --region $AWS_REGION

echo "=== Push Complete ==="
echo "Image URI: $ECR_URI:$VERSION"
echo "Latest URI: $ECR_URI:latest"
```

**Make executable and run:**
```bash
chmod +x push-to-ecr.sh
./push-to-ecr.sh
```

---

### PowerShell Script (push-to-ecr.ps1)

```powershell
$ErrorActionPreference = "Stop"

Write-Host "=== Push to AWS ECR (Mumbai Region) ===" -ForegroundColor Green

# Configuration
$AWS_REGION = "ap-south-1"
$AWS_ACCOUNT_ID = "587082268194"
$REPOSITORY_NAME = "erp-ventures"
$ECR_URI = "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPOSITORY_NAME"

# Get version
if (Test-Path version.txt) {
    $VERSION = Get-Content version.txt
} else {
    $VERSION = "latest"
}

Write-Host "AWS Account ID: $AWS_ACCOUNT_ID" -ForegroundColor Cyan
Write-Host "AWS Region: $AWS_REGION" -ForegroundColor Cyan
Write-Host "ECR URI: $ECR_URI" -ForegroundColor Cyan
Write-Host "Version: $VERSION" -ForegroundColor Cyan

# Step 1: Authenticate Docker to ECR
Write-Host "Authenticating to ECR..." -ForegroundColor Yellow
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

# Step 2: Tag image
Write-Host "Tagging image..." -ForegroundColor Yellow
docker tag "erp-ventures:$VERSION" "$ECR_URI:$VERSION"
docker tag "erp-ventures:latest" "$ECR_URI:latest"

# Step 3: Push image
Write-Host "Pushing image with version $VERSION..." -ForegroundColor Yellow
docker push "$ECR_URI:$VERSION"

Write-Host "Pushing image with tag latest..." -ForegroundColor Yellow
docker push "$ECR_URI:latest"

# Step 4: Verify
Write-Host "Verifying push..." -ForegroundColor Yellow
aws ecr describe-images --repository-name $REPOSITORY_NAME --region $AWS_REGION

Write-Host "=== Push Complete ===" -ForegroundColor Green
Write-Host "Image URI: $ECR_URI:$VERSION" -ForegroundColor Green
Write-Host "Latest URI: $ECR_URI:latest" -ForegroundColor Green
```

**Run:**
```powershell
.\push-to-ecr.ps1
```

---

## Combined Build and Push Script

### Git Bash (build-and-push.sh)

```bash
#!/bin/bash

set -e

echo "=== Build and Push to AWS ECR ==="

# Step 1: Build image with auto-increment version
./build.sh

# Configuration
AWS_REGION="ap-south-1"
AWS_ACCOUNT_ID="587082268194"
REPOSITORY_NAME="erp-ventures"
ECR_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPOSITORY_NAME"
VERSION=$(cat version.txt)

echo "Pushing version $VERSION to ECR..."

# Step 2: Authenticate
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Step 3: Tag and push
docker tag erp-ventures:$VERSION $ECR_URI:$VERSION
docker tag erp-ventures:latest $ECR_URI:latest

docker push $ECR_URI:$VERSION
docker push $ECR_URI:latest

echo "=== Complete ==="
echo "Image URI: $ECR_URI:$VERSION"
```

### PowerShell (build-and-push.ps1)

```powershell
$ErrorActionPreference = "Stop"

Write-Host "=== Build and Push to AWS ECR ===" -ForegroundColor Green

# Step 1: Build image with auto-increment version
.\build.ps1

# Configuration
$AWS_REGION = "ap-south-1"
$AWS_ACCOUNT_ID = "587082268194"
$REPOSITORY_NAME = "erp-ventures"
$ECR_URI = "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPOSITORY_NAME"
$VERSION = Get-Content version.txt

Write-Host "Pushing version $VERSION to ECR..." -ForegroundColor Yellow

# Step 2: Authenticate
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

# Step 3: Tag and push
docker tag "erp-ventures:$VERSION" "$ECR_URI:$VERSION"
docker tag "erp-ventures:latest" "$ECR_URI:latest"

docker push "$ECR_URI:$VERSION"
docker push "$ECR_URI:latest"

Write-Host "=== Complete ===" -ForegroundColor Green
Write-Host "Image URI: $ECR_URI:$VERSION" -ForegroundColor Green
```

---

## Verify in AWS Console

1. Go to: https://ap-south-1.console.aws.amazon.com/ecr/repositories
2. Click on **erp-ventures** repository
3. You should see your images with tags

---

## Quick Commands Reference

```bash
# Create repository (first time)
aws ecr create-repository --repository-name erp-ventures --region ap-south-1

# Authenticate (valid for 12 hours)
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 587082268194.dkr.ecr.ap-south-1.amazonaws.com

# Tag image
docker tag erp-ventures:1.0.0 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:1.0.0
docker tag erp-ventures:latest 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest

# Push image
docker push 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:1.0.0
docker push 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest

# List images
aws ecr describe-images --repository-name erp-ventures --region ap-south-1

# Delete specific image
aws ecr batch-delete-image --repository-name erp-ventures --image-ids imageTag=1.0.0 --region ap-south-1
```

---

## Your Final Image URIs

After push, your images will be available at:
- **Version tagged**: `587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:1.0.0`
- **Latest**: `587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest`

Use these URIs to pull the image on EC2 or deploy to ECS/EKS.

---

## Troubleshooting

### If authentication fails:
```bash
# Check AWS CLI is configured
aws sts get-caller-identity

# Should return your account ID: 587082268194
```

### If repository doesn't exist:
```bash
# Create it first
aws ecr create-repository --repository-name erp-ventures --region ap-south-1
```

### If push is slow:
- Mumbai region (ap-south-1) is closest to your location (Kerala)
- Push time depends on image size (~2.5GB) and internet speed
- Consider using AWS Direct Connect or faster internet for large images

---

## Next Steps

After successful push:
1. ✅ Pull image on EC2: `docker pull 587082268194.dkr.ecr.ap-south-1.amazonaws.com/erp-ventures:latest`
2. ✅ Deploy using docker-compose or ECS
3. ✅ Set up lifecycle policy for old images
4. ✅ Configure CI/CD pipeline
