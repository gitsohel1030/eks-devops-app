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
                env IMAGE_TAG=${IMAGE_TAG} envsubst < k8s/deployment.yaml > k8s/deployment_rendered.yaml
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
                sh "kubectl apply -f k8s/deployment_rendered.yaml"
            }
        }

        stage('Health Check') {
            steps {
                sh """
                    sleep 20
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://k8s-prodapp-webingre-fb76ccc10f-2038335722.ap-south-1.elb.amazonaws.com)
                    if ["$STATUS" != "200"]; then
                        echo "Health check failed"
                        exit 1
                    fi
                """
            }
        }

        stage('Verify Rollout') {
            steps {
                timeout(time: 2, unit: "MINUTES") {
                    sh "kubectl rollout status deployment/nginx -n prod-app"
            }
        }
    }

    post {
        failure {
            echo "Deployment failed, Rolling back."
            sh "kubectl rollout undo deployment/nginx -n prod-app"
        }

        success {
            echo "Deployment successful."
        }
    }
}