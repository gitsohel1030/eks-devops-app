pipeline {
  agent any

  environment {
    AWS_REGION     = "ap-south-1"
    CLUSTER_NAME   = "my-eks-cluster"
    ACCOUNT_ID     = "608827180555"
    REPO_NAME      = "web-app"
    ECR_REPO       = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}"
    // fallback populated in "Set IMAGE_TAG" stage if this doesn't expand as expected
    IMAGE_TAG      = "${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : ''}"
    IMAGE_EXISTS   = "false"
    DEPLOY_NEEDED  = "true"
    // K8S_NAMESPACE  = "prod-app"
    // K8S_DEPLOYMENT = "nginx"

    // App/K8s naming
    APP_NAME        = "nginx"          // base name for deployments
    K8S_NAMESPACE   = "prod-app"
    SERVICE_NAME    = "web-svc"        // the stable Service the ALB/Ingress points to

    // CURRENT_COLOR   = "blue"
    // TARGET_COLOR    = "green"

    SCALE_DOWN_OLD  = "true"           // set to "false" to keep old color running for quick rollback

  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Set IMAGE_TAG (fallback)') {
      when { expression { return !env.IMAGE_TAG || env.IMAGE_TAG.trim() == '' } }
      steps {
        script {
          env.IMAGE_TAG = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
          echo "Computed IMAGE_TAG=${env.IMAGE_TAG}"
        }
      }
    }

    stage('Check/Ensure ECR Repo + Tag') {
      steps {
        sh '''
          set -eu pipefail
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
        '''
        script {
          def m = readProperties file: '.image_exists.env'
          env.IMAGE_EXISTS = m['IMAGE_EXISTS']
          echo "IMAGE_EXISTS=${env.IMAGE_EXISTS}"
        }
      }
    }

    stage('Build Docker Image') {
      when { expression { env.IMAGE_EXISTS == 'false' } }
      steps { sh 'docker build -t ${ECR_REPO}:${IMAGE_TAG} .' }
    }

    stage('Login to ECR') {
      when { expression { env.IMAGE_EXISTS == 'false' } }
      steps {
        sh '''
          set -eu pipefail
          aws ecr get-login-password --region ${AWS_REGION} \
            | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
        '''
      }
    }

    stage('Push Image') {
      when { expression { env.IMAGE_EXISTS == 'false' } }
      steps { sh 'docker push ${ECR_REPO}:${IMAGE_TAG}' }
    }

    stage('Configure Kubectl') {
      steps { sh 'aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}' }
    }

    // stage('Check if Deploy Needed') {
    //   steps {
    //     sh '''
    //       set -eu pipefail

    //       DEPLOY_NEEDED=true
    //       if kubectl get deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} >/dev/null 2>&1; then
    //         CURRENT_IMAGE=$(kubectl get deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} -o jsonpath="{.spec.template.spec.containers[0].image}")
    //         echo "Current deployment image: ${CURRENT_IMAGE}"

    //         # Extract tag if present (repo:tag). If digest form, deploy anyway.
    //         if echo "${CURRENT_IMAGE}" | grep -q ':'; then
    //           CURRENT_TAG="${CURRENT_IMAGE##*:}"
    //         else
    //           CURRENT_TAG=""
    //         fi

    //         if [ "${CURRENT_TAG}" = "${IMAGE_TAG}" ]; then
    //           echo "Deployment already at desired tag: ${IMAGE_TAG}"
    //           DEPLOY_NEEDED=false
    //         else
    //           echo "Deployment tag (${CURRENT_TAG}) differs from desired (${IMAGE_TAG})."
    //         fi
    //       else
    //         echo "Deployment ${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} not found. Will deploy."
    //       fi

    //       echo "DEPLOY_NEEDED=${DEPLOY_NEEDED}" > .deploy_needed.env
    //     '''
    //     script {
    //       def p = readProperties file: '.deploy_needed.env'
    //       env.DEPLOY_NEEDED = p['DEPLOY_NEEDED']
    //       echo "DEPLOY_NEEDED=${env.DEPLOY_NEEDED}"
    //     }
    //   }
    // }

    stage('Decide Colors (current -> target)') {
      steps {
        sh '''
          set -eu
          # Read current color from Service selector; default to 'blue' if not set
          if kubectl get svc ${SERVICE_NAME} -n ${K8S_NAMESPACE} >/dev/null 2>&1; then
            CUR=$(kubectl get svc ${SERVICE_NAME} -n ${K8S_NAMESPACE} -o jsonpath="{.spec.selector.color}" 2>/dev/null || true)
          else
            CUR=""
          fi
          [ -z "${CUR}" ] && CUR="blue"
          if [ "${CUR}" = "blue" ]; then
            TAR="green"
          else
            TAR="blue"
          fi
          echo "CURRENT_COLOR=${CUR}" > .colors.env
          echo "TARGET_COLOR=${TAR}"  >> .colors.env
        '''
        script {
          def c = readProperties file: '.colors.env'
          env.CURRENT_COLOR = c['CURRENT_COLOR']
          env.TARGET_COLOR  = c['TARGET_COLOR']
          echo "CURRENT_COLOR=${env.CURRENT_COLOR}, TARGET_COLOR=${env.TARGET_COLOR}"
        }
      }
    }

    // stage('Update Manifest') {
    //   when { expression { env.DEPLOY_NEEDED == 'true' } }
    //   steps {
    //     sh '''
    //       set -eu
    //       IMAGE_TAG="${IMAGE_TAG}" envsubst < k8s/deployment.yaml > k8s/deployment_rendered.yaml
    //       echo "Rendered: k8s/deployment_rendered.yaml"
    //     '''
    //   }
    // }

    stage('Render Manifests (target color)') {
      steps {
        sh '''
          set -eu
          mkdir -p k8s/rendered

          # //Pick source deployment file based on TARGET_COLOR

          SRC_DEPLOY="k8s/deploy-${TARGET_COLOR}.yaml"

          # //Render deployment (inject image tag, names, namespace)

          APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" ECR_REPO="${ECR_REPO}" IMAGE_TAG="${IMAGE_TAG}" envsubst \
            < "${SRC_DEPLOY}" > "k8s/rendered/deploy-${TARGET_COLOR}.yaml"

          # //Render canary service pointing to TARGET_COLOR

          APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" SERVICE_NAME="${SERVICE_NAME}" COLOR="${TARGET_COLOR}" envsubst \
            < k8s/service-canary.yaml > "k8s/rendered/svc-canary-${TARGET_COLOR}.yaml"
        '''
      }
    }

    // stage('Deploy to EKS') {
    //   when { expression { env.DEPLOY_NEEDED == 'true' } }
    //   steps { sh 'kubectl apply -f k8s/deployment_rendered.yaml' }
    // }

    stage('Deploy Target Color & Wait') {
      steps {
        sh '''
          set -eu
          kubectl apply -f k8s/rendered/deploy-${TARGET_COLOR}.yaml
          kubectl rollout status deployment/${APP_NAME}-${TARGET_COLOR} -n ${K8S_NAMESPACE} --timeout=1m
        '''
      }
    }

    // stage('Verify Rollout') {
    //   when { expression { env.DEPLOY_NEEDED == 'true' } }
    //   steps {
    //     timeout(time: 2, unit: 'MINUTES') {
    //       sh 'kubectl rollout status deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE}'
    //     }
    //   }
    // }

    stage('Create Canary Service & Health Check (in-cluster)') {
      steps {
        sh '''
          set -eu
          kubectl apply -f k8s/rendered/svc-canary-${TARGET_COLOR}.yaml

          # Run a short-lived curl pod *inside the cluster* to hit the canary service
          kubectl delete pod tmp-curl -n ${K8S_NAMESPACE} --ignore-not-found
          kubectl run tmp-curl -n ${K8S_NAMESPACE} --image=curlimages/curl:8.10.1 --restart=Never --command -- \
            sh -c 'for i in $(seq 1 10); do code=$(curl -s -o /dev/null -w "%{http_code}" http://${SERVICE_NAME}-canary.${K8S_NAMESPACE}.svc.cluster.local:80/); echo "Try $i -> $code"; [ "$code" = "200" ] && exit 0; sleep 3; done; exit 1'
        '''
      }
    }

    // stage('Health Check') {
    //   when { expression { env.DEPLOY_NEEDED == 'true' } }
    //   steps {
    //     sh '''
    //       set -eu pipefail
    //       echo "Waiting for pods to stabilize..."
    //       sleep 20

    //       # Replace with your actual Ingress/ALB URL
    //       STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://k8s-prodapp-webingre-fb76ccc10f-1184307892.ap-south-1.elb.amazonaws.com)
    //       echo "HTTP Status: ${STATUS}"

    //       if [ "${STATUS}" != "200" ]; then
    //         echo "Health check failed"
    //         exit 1
    //       fi
    //       echo "Health check passed"
    //     '''
    //   }
    // }

    stage('Flip Service Selector to Target Color') {
      steps {
        sh '''
          set -eu
          kubectl patch svc ${SERVICE_NAME} -n ${K8S_NAMESPACE} \
            -p "{\"spec\": {\"selector\": {\"app\": \"${APP_NAME}\", \"version\": \"${TARGET_COLOR}\"}}}"
          echo "Switched ${SERVICE_NAME} selector to version=${TARGET_COLOR}"
        '''
      }
    }

    stage('Optional: Scale Down Old Color') {
      when { expression { env.SCALE_DOWN_OLD == 'true' } }
      steps {
        sh '''
          set -eu
          # Old color might not exist on first run; ignore if missing
          kubectl scale deployment/${APP_NAME}-${CURRENT_COLOR} -n ${K8S_NAMESPACE} --replicas=0 || true
        '''
      }
    }

    stage('Cleanup Canary Service') {
      steps {
        sh '''
          set -eu
          kubectl delete svc ${SERVICE_NAME}-canary -n ${K8S_NAMESPACE} --ignore-not-found
        '''
      }
    }

    stage('Done') {
      steps {
        echo "Blue/Green switch complete: ${CURRENT_COLOR} -> ${TARGET_COLOR}"
      }
    }
  }

  post {
    failure {
      script {
        echo "Pipeline failed — attempting to keep traffic on CURRENT_COLOR=${env.CURRENT_COLOR}."
      }
      // Best-effort: make sure stable service points back to CURRENT_COLOR
      sh '''set -eu
        kubectl patch svc ${SERVICE_NAME} -n ${K8S_NAMESPACE} \
          -p "{\"spec\": {\"selector\": {\"app\": \"${APP_NAME}\", \"version\": \"${CURRENT_COLOR}\"}}}" || true
        kubectl delete svc ${SERVICE_NAME}-canary -n ${K8S_NAMESPACE} --ignore-not-found || true
      '''
    }
    success {
      echo "Blue/Green deployment successful."
    }
  }
}