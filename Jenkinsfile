pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '587082268194'
        ECR_REPO = 'erp-ventures'
        // Construct the ECR URI
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
        IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPO}"
        // Versioning: 1.0.<BUILD_NUMBER>
        BUILD_VERSION = "1.0.${BUILD_NUMBER}"
    }

    stages {
        stage('Initialize') {
            steps {
                echo "Starting build for version: ${BUILD_VERSION}"
                echo "Repository: ${IMAGE_URI}"
            }
        }

        stage('Prepare') {
            steps {
                script {
                    // Check if apps.json exists
                    if (fileExists('apps.json')) {
                        echo "Encoding apps.json to Base64..."
                        
                        // Securely inject GitHub Token if present
                        withCredentials([string(credentialsId: 'github-access-token', variable: 'GITHUB_TOKEN')]) {
                            // Read file content
                            def appsJsonContent = readFile('apps.json')
                            
                            // Replace placeholder <GITHUB_TOKEN> with actual token
                            // We use a safe placeholder in apps.json like: https://<GITHUB_TOKEN>@github.com/org/repo
                            appsJsonContent = appsJsonContent.replace('<GITHUB_TOKEN>', env.GITHUB_TOKEN)
                            
                            // Encode to Base64
                            def appsJsonBase64 = appsJsonContent.bytes.encodeBase64().toString()
                            env.APPS_JSON_BASE64 = appsJsonBase64
                        }
                    } else {
                        error "apps.json file not found!"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker Image: ${IMAGE_URI}:${BUILD_VERSION}"
                // Note: We use the host's docker daemon mapped into the container
                sh """
                    docker build --no-cache \
                    --build-arg FRAPPE_PATH=https://github.com/casavaventures/frappe \
                    --build-arg FRAPPE_BRANCH=version-15 \
                    --build-arg APPS_JSON_BASE64=${APPS_JSON_BASE64} \
                    --tag ${IMAGE_URI}:${BUILD_VERSION} \
                    --tag ${IMAGE_URI}:latest \
                    --file images/layered/Containerfile .
                """
            }
        }

        stage('Login to ECR') {
            steps {
                echo "Logging into AWS ECR..."
                // Requires AWS credentials with ID 'aws-ecr-credentials' in Jenkins
                withCredentials([usernamePassword(credentialsId: 'aws-ecr-credentials', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                }
            }
        }

        stage('Push to ECR') {
            steps {
                echo "Pushing image to ECR..."
                sh """
                    docker push ${IMAGE_URI}:${BUILD_VERSION}
                    docker push ${IMAGE_URI}:latest
                """
            }
        }
    }

    post {
        always {
            echo "Cleaning up..."
            // Remove the local images to save space on the Jenkins host
            sh """
                docker rmi ${IMAGE_URI}:${BUILD_VERSION} || true
                docker rmi ${IMAGE_URI}:latest || true
            """
        }
        success {
            echo "Build and Push Successful: ${IMAGE_URI}:${BUILD_VERSION}"
        }
        failure {
            echo "Build Failed"
        }
    }
}
