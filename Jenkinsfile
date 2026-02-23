pipeline {
  agent any

  environment {
    AWS_REGION   = "ap-south-1"
    CLUSTER_NAME = "my-eks-cluster"
    ACCOUNT_ID   = "608827180555"
    REPO_NAME    = "web-app"
    ECR_REPO     = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}"
    // fallback populated in "Set IMAGE_TAG" stage if this doesn't expand as expected
    IMAGE_TAG    = "${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : ''}"
    IMAGE_EXISTS = "false"
    DEPLOY_NEEDED = "true"
    K8S_NAMESPACE = "prod-app"
    K8S_DEPLOYMENT = "nginx"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set IMAGE_TAG (fallback)') {
      when {
        expression { return !env.IMAGE_TAG || env.IMAGE_TAG.trim() == '' }
      }
      steps {
        script {
          // Compute a short tag from git if not set by env expansion.
          env.IMAGE_TAG = sh(
            script: 'git rev-parse --short=7 HEAD',
            returnStdout: true
          ).trim()
          echo "Computed IMAGE_TAG=${env.IMAGE_TAG}"
        }
      }
    }

    stage('Check/Ensure ECR Repo + Tag') {
      steps {
        sh """
          set -euo pipefail
          echo "Ensuring ECR repository ${REPO_NAME} exists (region: ${AWS_REGION})..."

          if ! aws ecr describe-repositories \
                --repository-names "${REPO_NAME}" \
                --region "${AWS_REGION}" >/dev/null 2>&1; then
            echo "ECR repo ${REPO_NAME} not found. Creating..."

            aws ecr create-repository \
              --repository-name "${REPO_NAME}" \
              --region "${AWS_REGION}" >/dev/null
            echo "Created ECR repository: ${REPO_NAME}"

          else
            echo "ECR repository ${REPO_NAME} exists."
          fi

          echo "Checking if image tag exists: ${REPO_NAME}:${IMAGE_TAG}"
          if aws ecr describe-images \
                --region "${AWS_REGION}" \
                --repository-name "${REPO_NAME}" \
                --image-ids imageTag="${IMAGE_TAG}" >/dev/null 2>&1; then

            echo "IMAGE_EXISTS=true" > .image_exists.env
            echo "Image tag already present in ECR."

          else
            echo "IMAGE_EXISTS=false" > .image_exists.env
            echo "Image tag not present in ECR."
          fi
        """
        script {
          def m = readProperties file: '.image_exists.env'
          env.IMAGE_EXISTS = m['IMAGE_EXISTS']
          echo "IMAGE_EXISTS=${env.IMAGE_EXISTS}"
        }
      }
    }

    stage('Build Docker Image') {
      when { expression { env.IMAGE_EXISTS == 'false' } }
      steps {
        sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
      }
    }

    stage('Login to ECR') {
      when { expression { env.IMAGE_EXISTS == 'false' } }
      steps {
        sh """
          set -euo pipefail
          aws ecr get-login-password --region ${AWS_REGION} \
            | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
        """
      }
    }

    stage('Push Image') {
      when { expression { env.IMAGE_EXISTS == 'false' } }
      steps {
        sh "docker push ${ECR_REPO}:${IMAGE_TAG}"
      }
    }

    stage('Configure Kubectl') {
      steps {
        sh "aws eks update-kubeconfig --region \${AWS_REGION} --name \${CLUSTER_NAME}"
      }
    }

    stage('Check if Deploy Needed') {
      steps {
        sh """
          set -euo pipefail

          DEPLOY_NEEDED=true
          if kubectl get deployment/"${K8S_DEPLOYMENT}" -n \${K8S_NAMESPACE} >/dev/null 2>&1; then
            CURRENT_IMAGE=$(kubectl get deployment/\${K8S_DEPLOYMENT} -n \${K8S_NAMESPACE} -o jsonpath="{.spec.template.spec.containers[0].image}")
            echo "Current deployment image: \${CURRENT_IMAGE}"
            
            if echo "\${CURRENT_IMAGE}" | grep -q ':'; then
              CURRENT_TAG="\${CURRENT_IMAGE##*:}"
            else
              CURRENT_TAG=""  # no tag to compare; assume deploy needed
            fi

            if [ "\${CURRENT_TAG}" = "\${IMAGE_TAG}" ]; then
              echo "Deployment already at desired tag: \${IMAGE_TAG}"
              DEPLOY_NEEDED=false
            else
              echo "Deployment tag (\${CURRENT_TAG}) differs from desired (\${IMAGE_TAG})."
            fi
          else
            echo "Deployment \${K8S_DEPLOYMENT} -n \${K8S_NAMESPACE} not found. Will deploy."
          fi

          echo "DEPLOY_NEEDED=\${DEPLOY_NEEDED}" > .deploy_needed.env
        """
        script {
          def p = readProperties file: '.deploy_needed.env'
          env.DEPLOY_NEEDED = p['DEPLOY_NEEDED']
          echo "DEPLOY_NEEDED=\${env.DEPLOY_NEEDED}"
        }
      }
    }

    stage('Update Manifest') {
      when { expression { env.DEPLOY_NEEDED == 'true' } }
      steps {
        sh """
          set -eu
          # Render manifest by substituting IMAGE_TAG inside template
          IMAGE_TAG="\${IMAGE_TAG}" envsubst < k8s/deployment.yaml > k8s/deployment_rendered.yaml
          echo "Rendered: k8s/deployment_rendered.yaml"
        """
      }
    }

    stage('Deploy to EKS') {
      when { expression { env.DEPLOY_NEEDED == 'true' } }
      steps {
        sh "kubectl apply -f k8s/deployment_rendered.yaml"
      }
    }

    stage('Verify Rollout') {
      when { expression { env.DEPLOY_NEEDED == 'true' } }
      steps {
        timeout(time: 2, unit: 'MINUTES') {
          sh "kubectl rollout status deployment/\${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE}"
        }
      }
    }

    stage('Health Check') {
      when { expression { env.DEPLOY_NEEDED == 'true' } }
      steps {
        sh """
          set -euo pipefail
          echo "Waiting for pods to stabilize..."
          sleep 20

          # Replace this with your service/ingress URL or a cluster-internal check
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://k8s-prodapp-webingre-fb76ccc10f-2038335722.ap-south-1.elb.amazonaws.com)
          echo "HTTP Status: \${STATUS}"

          if [ "\${STATUS}" != "200" ]; then
            echo "Health check failed"
            exit 1
          fi
          echo "Health check passed"
        """
      }
    }

    stage('Skip Notice (no deploy)') {
      when { expression { env.DEPLOY_NEEDED == 'false' } }
      steps {
        echo "Skipping Update/Deploy/Health/Rollout — deployment already runs tag \${IMAGE_TAG}."
      }
    }

    stage('Skip Notice (no push)') {
      when { expression { env.IMAGE_EXISTS == 'true' } }
      steps {
        echo "Skipping docker build/login/push — image ${REPO_NAME}:${IMAGE_TAG} already exists in ECR."
      }
    }
  }

  post {
    failure {
      echo "Deployment failed, attempting rollback..."
      sh "kubectl rollout undo deployment/\${K8S_DEPLOYMENT} -n \${K8S_NAMESPACE} || true"
    }
    success {
      echo "Pipeline successful..."
    }
  }
}