pipeline {
    agent any

    environment {
        // Jenkins credentials
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-creds')            // DockerHub
        GIT_CREDENTIALS = credentials('GitAccess')                         // GitHub access
        ARGOCD_CREDENTIALS = credentials('argocd')                         // ArgoCD access

        // AWS and DockerHub details
        REGION = 'ap-south-1'
        IMAGE_TAG = "${BUILD_NUMBER}"
        AWS_ACCOUNT_ID = '439110395780'
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/my-repo"
        DOCKERHUB_REPO = 'avikbhattacharya056/my-calculator-image'
        ARGOCD_SERVER = '13.233.233.91:30493'
        GITHUB_REPO = 'https://github.com/AvikBhattacharya-Secops/complete-project-all.git' // GitHub Repo URL
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Cloning from GitHub main branch..."
                    // Checkout the repo using the Git plugin with credentials
                    checkout scm
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
                    sh "docker tag ${DOCKERHUB_REPO}:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    echo "Logging in and pushing to DockerHub..."
                    sh """
                        echo "${DOCKERHUB_CREDENTIALS_PSW}" | docker login -u "${DOCKERHUB_CREDENTIALS_USR}" --password-stdin
                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                        docker logout
                    """
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                script {
                    echo "Logging in and pushing to AWS ECR..."
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                        sh """
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                            docker push ${ECR_REPO}:${IMAGE_TAG}
                            docker logout
                        """
                    }
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                script {
                    echo "Updating Helm values.yaml..."
                    sh """
                        sed -i 's|repository:.*|repository: ${DOCKERHUB_REPO}|' helm/values.yaml
                        sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' helm/values.yaml
                    """
                }
            }
        }

        stage('Push Helm Changes') {
            steps {
                script {
                    echo "Committing and pushing Helm changes to GitHub..."
                    withCredentials([usernamePassword(credentialsId: 'GitAccess', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASSWORD')]) {
                        // URL encode the GitHub username to handle special characters like '@'
                        def encodedUsername = GIT_USER.replace('@', '%40')

                        sh """
                            git config user.email 'ci@jenkins.com'
                            git config user.name 'Jenkins CI'

                            # Ensure we are on the 'main' branch before committing and pushing
                            git checkout main || git checkout -b main  # Checkout main or create if doesn't exist

                            git add helm/values.yaml
                            git diff --cached --quiet || git commit -m 'Update image tag to ${IMAGE_TAG}'

                            # Push the changes using the GitHub credentials for authentication
                            git push https://${encodedUsername}:${GIT_PASSWORD}@github.com/AvikBhattacharya-Secops/complete-project-all.git main
                        """
                    }
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                script {
                    echo "Triggering ArgoCD sync..."
                    withCredentials([usernamePassword(credentialsId: 'argocd', usernameVariable: 'ARGOCD_USER', passwordVariable: 'ARGOCD_PASSWORD')]) {
                        sh """
                            argocd login ${ARGOCD_SERVER} --username $ARGOCD_USER --password $ARGOCD_PASSWORD --insecure || echo 'ArgoCD login failed or already logged in'
                            argocd app sync calculator-app || echo 'ArgoCD sync failed or app not found'
                        """
                    }
                }
            }
        }
    }

    post {
        failure {
            echo 'Build failed. Check logs for details.'
        }
        always {
            cleanWs()
        }
    }
}
