pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        CLUSTER_NAME = "my-eks-cluster"
        ACCOUNT_ID = "608827180555"
        ECR_REPO = "${ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/web-app"
        IMAGE_TAG = "${env.GIT_COMMIT}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Login to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_REGION} \
                | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                """
            }
        }

        stage('Push Image') {
            steps {
                sh "docker push ${ECR_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Update Manifest') {
            steps{
                sh """
                sed -i 's|IMAGE_TAG|${IMAGE_TAG}g' k8s/deployment.yaml
                """
            }
        }

        stage('Configure Kubectl') {
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
            }
        }

        stage('Deploy to EKS') {
            steps{
                sh "kubectl apply -f k8s/"
            }
        }

        stage('Verify Rollout') {
            steps{
                sh "kubectl rollout status deployment/web-app -n prod-app"
            }
        }
    }

    post {
        failure {
            echo "Deployment failed."
        }

        success {
            echo "Deployment successful."
        }
    }
}