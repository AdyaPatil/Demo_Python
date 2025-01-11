pipeline {
    agent any

    environment {
        ECR_REGISTRY = "<aws_account_id>.dkr.ecr.<region>.amazonaws.com"
        ECR_REPOSITORY = "demo-python"
        IMAGE_TAG = "latest"
        EKS_CLUSTER = "<your-eks-cluster-name>"
        K8S_NAMESPACE = "default"
        AWS_REGION = "<region>"
        DOCKER_CREDENTIALS = credentials('docker-credentials')  // Jenkins credentials for Docker login
        AWS_CREDENTIALS = credentials('aws-credentials')  // Jenkins credentials for AWS access
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Login to ECR') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker build -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    sh """
                        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    // Update the Kubernetes deployment with the new Docker image
                    sh """
                        sed -i "s|image: .*|image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}|" demo-python-deployment.yaml
                    """
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                            aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER}
                            kubectl apply -f demo-python-deployment.yaml
                            kubectl apply -f demo-python-service.yaml
                        """
                    }
                }
            }
        }

        stage('Sync with ArgoCD') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'argocd-auth-token', variable: 'ARGOCD_AUTH_TOKEN')]) {
                        sh """
                            argocd login <argo_cd_server> --username admin --password ${ARGOCD_AUTH_TOKEN} --insecure
                            argocd app sync demo-python-app --grpc-web
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment was successful'
        }
        failure {
            echo 'Deployment failed'
        }
    }
}
