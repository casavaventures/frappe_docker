# Jenkins CI/CD Setup Guide for Frappe Docker

This guide details how to set up a containerized Jenkins environment to automate building and pushing Frappe Docker images to AWS ECR.

## 1. Prerequisites
- Docker & Docker Compose installed on your host machine (Windows/Linux/Mac).
- AWS Account with ECR Repository created (`erp-ventures` in `ap-south-1`).
- GitHub Account with access to the repositories.
- `frappe_docker` repository cloned locally.

## 2. Project Structure
Ensure your `frappe_docker` directory has the following structure:

```
frappe_docker/
├── apps.json                  # Your custom apps definition
├── Jenkinsfile                # CI Pipeline definition
├── jenkins/
│   ├── docker-compose.yml     # Jenkins service definition
│   └── Dockerfile             # Custom Jenkins image definition
└── ... (other files)
```

### Create `apps.json`
If you haven't already, create `apps.json` in the root of `frappe_docker`. Use `<GITHUB_TOKEN>` as a placeholder for private repositories.

```json
[
  {
    "url": "https://github.com/casavaventures/erpnext",
    "branch": "version-15"
  },
  {
    "url": "https://<GITHUB_TOKEN>@github.com/casavaventures/shopbridge.git",
    "branch": "release"
  }
]
```

## 3. Start Jenkins

We run Jenkins in a Docker container that has access to the host's Docker socket, allowing it to spawn sibling containers for building images.

1.  Open a terminal in `frappe_docker/jenkins`.
2.  Run the following command to build and start Jenkins:

    ```bash
    docker-compose up --build -d
    ```

3.  Wait for Jenkins to start (it may take a minute). Check logs if needed: `docker logs jenkins`

## 4. Initial Jenkins Configuration

1.  **Unlock Jenkins**:
    - Retrieve the initial admin password:
      ```bash
      docker logs jenkins
      ```
    - Open your browser to `http://localhost:8080`.
    - Paste the password.

2.  **Install Plugins**:
    - Choose **"Install suggested plugins"**.
    - After installation completes, go to **Dashboard > Manage Jenkins > Plugins > Available plugins**.
    - Search for and install:
        - `Docker Pipeline`
        - `Pipeline: AWS Steps` (optional helper)

3.  **Create Admin User**: Follow the prompts to set up your admin account.

## 5. Configure Credentials

Jenkins needs authorization to pull from your private GitHub repos and push to AWS ECR.

### AWS Credentials

You have two options for handling AWS credentials. **Option 1 (Stored Credentials)** is enabled by default for automation.

#### Option 1: Stored Credentials (Automated) - Default
Use this for fully automated builds (no manual input).
1.  Go to **Dashboard > Manage Jenkins > Credentials > System > Global credentials (unrestricted)**.
2.  Click **+ Add Credentials**.
3.  **Kind**: Username with password.
4.  **Username**: `AWS_ACCESS_KEY_ID` (Your AWS Access Key).
5.  **Password**: `AWS_SECRET_ACCESS_KEY` (Your AWS Secret Key).
6.  **ID**: `aws-ecr-credentials` (**Crucial**: Must match this exact ID).
7.  Click **Create**.
**Need Keys?** [Generate here](https://console.aws.amazon.com/iam/home?#/security_credentials).

#### Option 2: Interactive Input (Manual)
If you prefer NOT to store credentials and enter them manually for every build:
1.  Edit `Jenkinsfile`.
2.  Comment out the "Option 2" block and uncomment the "Option 1" block.
3.  The pipeline will then pause and ask for credentials during the build.

### GitHub Token (for Private Repos)
1.  Click **+ Add Credentials** again.
2.  **Kind**: Secret text
3.  **Scope**: Global
4.  **Secret**: Your GitHub Personal Access Token (PAT).
    - *Note*: Ensure the PAT has `repo` scope.
5.  **ID**: `github-access-token` (**Crucial**: Must match this exact ID)
6.  Click **Create**.

## 6. Configure Global Variables

To keep your `Jenkinsfile` generic and secure, we will set the project-specific values as global environment variables.

1.  Go to **Dashboard > Manage Jenkins > System**.
2.  Scroll down to **Global properties**.
3.  Check **Environment variables**.
4.  Click **Add** for each of the following:
    - **Name**: `AWS_DEFAULT_REGION` -> **Value**: `ap-south-1`
    - **Name**: `AWS_ACCOUNT_ID` -> **Value**: `587082268194`
    - **Name**: `ECR_REPO` -> **Value**: `erp-ventures`
5.  Click **Save**.

## 7. Create Pipeline Job

1.  Go to **Dashboard > New Item**.
2.  Enter a name (e.g., `frappe-docker-build`).
3.  Select **Pipeline** and click **OK**.
4.  Scroll down to the **Pipeline** section.
5.  **Definition**: `Pipeline script from SCM`.
6.  **SCM**: `Git`.
7.  **Repository URL**: `https://github.com/casavaventures/frappe_docker.git`
    - *Note*: Since the repo is private, you may need to add your GitHub credentials here (User/Pass) just for Jenkins to clone the `Jenkinsfile`. This is different from the `github-access-token` used inside the pipeline.
8.  **Branch Specifier**: `*/main`
9.  **Script Path**: `Jenkinsfile`
10. Click **Save**.

## 8. Run the Build

1.  Click **Build Now** on the job page.
2.  Monitor progress by clicking the build number (`#1`) > **Console Output**.

### What Happens During the Build?
1.  **Initialize**: Sets version.
2.  **Prepare**: Prepares apps.
3.  **Build**: Builds Docker image.
4.  **Login**: Authenticates with AWS ECR using stored credentials automatically.
5.  **Push**: Pushes to ECR utilizing the credentials.
6.  **Cleanup**: Removes local images.

## Troubleshooting

- **Socket Permission Denied**: If Jenkins fails to connect to Docker (`permission denied`), check `docker-compose.yml`. We use `user: root` to bypass this, but ideally, you should add the jenkins user to the docker group on the host if running in a production linux environment. for Windows Docker Desktop, this usually works out of the box.
- **Login Failed**: Verify AWS Credentials ID is exactly `aws-ecr-credentials`.
- **Private Repo Clone Failed**: Verify GitHub Token ID is exactly `github-access-token`.
