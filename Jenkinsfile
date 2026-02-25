pipeline {
  agent any

  environment {
    AWS_REGION     = "ap-south-1"
    CLUSTER_NAME   = "my-eks-cluster"
    ACCOUNT_ID     = "608827180555"
    REPO_NAME      = "web-app"
    ECR_REPO       = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}"

    IMAGE_TAG      = "${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : ''}"
    IMAGE_EXISTS   = "false"
    DEPLOY_NEEDED  = "true"
    K8S_NAMESPACE  = "prod-app"

    // App/K8s naming
    APP_NAME        = "web-app"          // base name for deployments
    SERVICE_NAME    = "web-svc"        // the stable Service the ALB/Ingress points to

    CURRENT_COLOR   = ""
    TARGET_COLOR    = ""

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

        
    stage('Ensure Namespace') {
      steps {
        sh '''
          set -eu pipefail
          kubectl get ns ${K8S_NAMESPACE} >/dev/null 2>&1 || kubectl create ns ${K8S_NAMESPACE}
        '''
      }
    }

    // stage('Deploy MySQL (dev)') {
    //   steps {
    //     withCredentials([
    //       string(credentialsId: 'mysql-root-password', variable: 'MYSQL_ROOT_PASSWORD'),
    //       string(credentialsId: 'mysql-app-password',  variable: 'MYSQL_PASSWORD')
    //     ]) {
    //       sh '''
    //         set -eu pipefail

          
    //         # ConfigMap, PVC, Service
    //         K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/db-dev/configmap.yaml | kubectl apply -f -
    //         K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/db-dev/pvc.yaml      | kubectl apply -f -
    //         K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/db-dev/service.yaml  | kubectl apply -f -

    //         # Secret from Jenkins (dont store in Git)
    //         kubectl create secret generic mysql-secret \
    //           -n ${K8S_NAMESPACE} \
    //           --from-literal=MYSQL_ROOT_PASSWORD="${MYSQL_ROOT_PASSWORD}" \
    //           --from-literal=MYSQL_PASSWORD="${MYSQL_PASSWORD}" \
    //           --dry-run=client -o yaml | kubectl apply -f -

    //         # Deployment + PDB
    //         K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/db-dev/deployment.yaml | kubectl apply -f -
    //         K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/db-dev/pdb.yaml        | kubectl apply -f -

    //         # Wait until ready
    //         kubectl rollout status deploy/mysql -n ${K8S_NAMESPACE} --timeout=3m
    //       '''
    //     }
    //   }
    // }


    stage('Apply Services + HPAs') {
      steps {
        sh '''
          set -eu pipefail
          mkdir -p k8s/base/blue-green/rendered
          # services
          APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/blue-green/service-blue.yaml  > k8s/base/blue-green/rendered/service-blue.yaml
          APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/blue-green/service-green.yaml > k8s/base/blue-green/rendered/service-green.yaml
          kubectl apply -f k8s/base/blue-green/rendered/service-blue.yaml
          kubectl apply -f k8s/base/blue-green/rendered/service-green.yaml

          # hpas (optional)
          APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/blue-green/hpa-blue.yaml  > k8s/base/blue-green/rendered/hpa-blue.yaml
          APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/blue-green/hpa-green.yaml > k8s/base/blue-green/rendered/hpa-green.yaml
          kubectl apply -f k8s/base/blue-green/rendered/hpa-blue.yaml
          kubectl apply -f k8s/base/blue-green/rendered/hpa-green.yaml
        '''
      }
    }


    stage('Determine TARGET color') {
      steps {
        script {
          def activeColor = sh(
            script: '''
              set -eu

              # Check ready replicas of blue
              BLUE=$(kubectl get deploy ${APP_NAME}-blue -n ${K8S_NAMESPACE} -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0)
              BLUE=${BLUE:-0}

              # Check ready replicas of green
              GREEN=$(kubectl get deploy ${APP_NAME}-green -n ${K8S_NAMESPACE} -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0)
              GREEN=${GREEN:-0}

              if [ "$BLUE" -gt 0 ]; then
                echo blue
              elif [ "$GREEN" -gt 0 ]; then
                echo green
              else
                # first deployment ever → assume blue is active
                echo blue
              fi
            ''',
            returnStdout: true
          ).trim()

          env.CURRENT_COLOR = activeColor
          env.TARGET_COLOR  = (activeColor == "blue") ? "green" : "blue"

          echo "Active color detected: ${env.CURRENT_COLOR}"
          echo "Will deploy target color: ${env.TARGET_COLOR}"
        }
      }
    }

    stage('Deploy TARGET color') {
      steps {
        sh '''
          set -eu pipefail
          mkdir -p k8s/base/blue-green/rendered
          
          if [ "${TARGET_COLOR}" = "green" ]; then
            APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" ECR_REPO="${ECR_REPO}" IMAGE_TAG="${IMAGE_TAG}" envsubst \
              < k8s/base/blue-green/blue-deployment.yaml > k8s/base/blue-green/rendered/deploy-blue.yaml
            kubectl apply -f k8s/base/blue-green/rendered/deploy-blue.yaml
            kubectl rollout status deployment/${APP_NAME}-blue -n ${K8S_NAMESPACE} --timeout=2m

            else
            APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" ECR_REPO="${ECR_REPO}" IMAGE_TAG="${IMAGE_TAG}" envsubst \
              < k8s/base/blue-green/green-deployment.yaml > k8s/base/blue-green/rendered/deploy-green.yaml
            kubectl apply -f k8s/base/blue-green/rendered/deploy-green.yaml
            kubectl rollout status deployment/${APP_NAME}-green -n ${K8S_NAMESPACE} --timeout=2m
          fi
        '''
      }
    }

    stage('ALB Weighted Shift (Declarative Apply)') {
      steps {
        sh '''
          set -eu pipefail
          mkdir -p k8s/base/blue-green/rendered

          # Define weight schedule toward TARGET_COLOR

          if [ "${TARGET_COLOR}" = "green" ]; then
            STEPS="90:10 50:50 0:100"
          else
            STEPS="10:90 50:50 100:0"
          fi

          for W in $STEPS; do
            WB="${W%%:*}"; WG="${W##*:}"
            echo "Applying weights blue=${WB}, green=${WG}"

            K8S_NAMESPACE="${K8S_NAMESPACE}" WEIGHT_BLUE="${WB}" WEIGHT_GREEN="${WG}" envsubst \
              < k8s/base/blue-green/ingress.yaml > k8s/base/blue-green/rendered/ingress.yaml

            kubectl apply -f k8s/base/blue-green/rendered/ingress.yaml

            #echo "Waiting 25s for ALB to pick up new weights..."
            #sleep 25

            # Example (once I have the ALB DNS):
            #CODE=$(curl -s -o /dev/null -w "%{http_code}" "http://k8s-prodapp-webappin-cd99108c4d-1318376321.ap-south-1.elb.amazonaws.com/" || true)
            #echo "ALB probe -> HTTP ${CODE}"
          #done
        '''
      }
    }


    stage('Optional: Scale down old color') {
      when { expression { return true } } // set to false to keep for fast rollback
      steps {
        sh '''
          set -eu pipefail
          if [ "${TARGET_COLOR}" = "green" ]; then
            kubectl scale deployment/${APP_NAME}-blue -n ${K8S_NAMESPACE} --replicas=0 || true
          else
            kubectl scale deployment/${APP_NAME}-green -n ${K8S_NAMESPACE} --replicas=0 || true
          fi
        '''
      }
    }
  }


post {
    failure {
      echo "Pipeline failed; attempting to revert to previous weights (100% old color)."
      sh '''
        set -eu pipefail

        mkdir -p k8s/base/blue-green/rendered
        
        if [ "${TARGET_COLOR}" = "green" ]; then
          # revert to BLUE=100, GREEN=0
          WEIGHT_BLUE=100
          WEIGHT_GREEN=0
        else
          # revert to BLUE=0, GREEN=100
          WEIGHT_BLUE=0
          WEIGHT_GREEN=100
        fi

        # Re-render ingress without host/cert
        K8S_NAMESPACE="${K8S_NAMESPACE}" WEIGHT_BLUE="${WEIGHT_BLUE}" WEIGHT_GREEN="${WEIGHT_GREEN}" envsubst \
          < k8s/base/blue-green/ingress.yaml > k8s/base/blue-green/rendered/ingress-revert.yaml
        kubectl apply -f k8s/base/blue-green/rendered/ingress-revert.yaml
      '''
    }
    success { echo "ALB-based blue/green rollout complete..." }
  }
}
