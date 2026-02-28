pipeline {
  agent any

  environment {
    AWS_REGION     = "ap-south-1"
    CLUSTER_NAME   = "my-eks-cluster-1030"
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

    // DEPRECATED
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


    // stage('Apply Services + HPAs') {
    //   steps {
    //     sh '''
    //       set -eu pipefail
    //       mkdir -p k8s/base/blue-green/rendered
    //       # services
    //       APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/service-blue.yaml  > k8s/base/blue-green/rendered/service-blue.yaml
    //       APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/service-green.yaml > k8s/base/blue-green/rendered/service-green.yaml
    //       kubectl apply -f k8s/base/blue-green/rendered/service-blue.yaml
    //       kubectl apply -f k8s/base/blue-green/rendered/service-green.yaml

    //       # hpas (optional)
    //       APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/blue-green/hpa-blue.yaml  > k8s/base/blue-green/rendered/hpa-blue.yaml
    //       APP_NAME="${APP_NAME}" K8S_NAMESPACE="${K8S_NAMESPACE}" envsubst < k8s/base/blue-green/hpa-green.yaml > k8s/base/blue-green/rendered/hpa-green.yaml
    //       kubectl apply -f k8s/base/blue-green/rendered/hpa-blue.yaml
    //       kubectl apply -f k8s/base/blue-green/rendered/hpa-green.yaml
    //     '''
    //   }
    // }


    stage('Determine TARGET color') {
      steps {
        script {
          def active = sh(
            label: 'Detect active color',
            returnStdout: true,
            script: '''
                bash -lc "
                  set -Eeuo pipefail

                  BLUE=\\$(kubectl get deploy \${APP_NAME}-blue  -n \${K8S_NAMESPACE} -o go-template='{{or .status.readyReplicas 0}}' 2>/dev/null || echo 0)
                  GREEN=\$(kubectl get deploy \${APP_NAME}-green -n \${K8S_NAMESPACE} -o go-template='{{or .status.readyReplicas 0}}' 2>/dev/null || echo 0)

                  # Debug to stderr (won't pollute stdout capture)
                  echo \\"[detect] BLUE=\$BLUE GREEN=\$GREEN\\" 1>&2

                  if [ \\"\$BLUE\\" -gt 0 ]; then
                    echo blue
                  elif [ \\"\$GREEN\\" -gt 0 ]; then
                    echo green
                  else
                    echo none
                  fi
                "
                '''
          ).trim()

          if (active == "blue") {
            env.CURRENT_COLOR = "blue"
            env.TARGET_COLOR  = "green"
          } else if (active == "green") {
            env.CURRENT_COLOR = "green"
            env.TARGET_COLOR  = "blue"
          } else {
            // 'none' or unexpected → default first deploy to blue
            env.CURRENT_COLOR = "none"
            env.TARGET_COLOR  = "blue"
          }

          echo "Active Color: ${env.CURRENT_COLOR}"
          echo "Target Color: ${env.TARGET_COLOR}"
        }
      }
    }

    stage('Build & Apply Kustomize Overlay (commit-driven weights)') {
      steps {
        sh label: 'Build and Apply Kustomize', script: '''
            bash -lc '
              set -eu pipefail

              OUT="k8s/.out/prod"
              rm -rf "${OUT}"
              mkdir -p "${OUT}"

              # Copy overlay to a working dir we can mutate
              cp -R k8s/overlays/prod/* "${OUT}/"

              # Optional: if you persist colors in a file, load them
              if [[ -f .colors.env ]]; then
                set -a
                # shellcheck disable=SC1091
                source ./.colors.env
                set +a
              fi

              # Fallback: compute TARGET_COLOR if still missing (first run / env loss)
              if [[ -z "${TARGET_COLOR:-}" ]]; then
                BLUE=$(kubectl get deploy "${APP_NAME}-blue"  -n "${K8S_NAMESPACE}" -o jsonpath="{.status.readyReplicas}" 2>/dev/null || echo 0);  BLUE=${BLUE:-0}
                GREEN=$(kubectl get deploy "${APP_NAME}-green" -n "${K8S_NAMESPACE}" -o jsonpath="{.status.readyReplicas}" 2>/dev/null || echo 0); GREEN=${GREEN:-0}
                if [[ "$BLUE" -gt 0 ]]; then
                  CURRENT_COLOR="blue"; TARGET_COLOR="green"
                elif [[ "$GREEN" -gt 0 ]]; then
                  CURRENT_COLOR="green"; TARGET_COLOR="blue"
                else
                  CURRENT_COLOR="none"; TARGET_COLOR="blue"
                fi
                printf "CURRENT_COLOR=%s\nTARGET_COLOR=%s\n" "$CURRENT_COLOR" "$TARGET_COLOR" > .colors.env
              fi

              # Choose correct patch by TARGET_COLOR
              PATCH_FILE="${OUT}/patch-blue-image.yaml"
              if [[ "${TARGET_COLOR:-}" == "green" ]]; then
                sed -i "s|patch-blue-image.yaml|patch-green-image.yaml|" "${OUT}/kustomization.yaml"
                PATCH_FILE="${OUT}/patch-green-image.yaml"
              fi

              # Stamp the image tag placeholder in the selected patch
              sed -i "s|__IMAGE_TAG__|${IMAGE_TAG}|g" "${PATCH_FILE}" || true

              echo "=== Kustomize build (preview) ==="
              kubectl kustomize "${OUT}" | head -n 200 || true

              echo "=== Diff against cluster (informational) ==="
              kubectl diff -k "${OUT}" || true

              echo "=== Apply overlay ==="
              kubectl apply -k "${OUT}"

              echo "=== Wait for TARGET rollout ==="
              kubectl rollout status "deploy/${APP_NAME}-${TARGET_COLOR}" -n "${K8S_NAMESPACE}" --timeout=2m

              # Optional: mark change cause on the deployment
              kubectl annotate "deploy/${APP_NAME}-${TARGET_COLOR}" \
                -n "${K8S_NAMESPACE}" kubernetes.io/change-cause="Deploy ${IMAGE_TAG} to ${TARGET_COLOR}" --overwrite || true

              echo "=== Pods (post-deploy) ==="
              kubectl get pods -n "${K8S_NAMESPACE}" -l app="${APP_NAME}" -o wide
            '
            '''
      }
    }

    stage('Show Traffic Weights (from Git overlay)') {
      steps {
        sh '''set -eu pipefail

            ING="${APP_NAME}-ingress"
            # Read the forward action annotation from the live Ingress (commit-driven)
            ANN=$(kubectl get ing/${ING} -n ${K8S_NAMESPACE} -o jsonpath="{.metadata.annotations.alb\\.ingress\\.kubernetes\\.io/actions\\.forward-blue-green}" || true)
            echo "Current ALB forward weights (live): ${ANN}"
        '''
      }
    }


    
    stage('Optional: Scale down OLD color') {
      when { expression { return env.SCALE_DOWN_OLD == 'true' } }
      steps {
        sh '''set -eu pipefail
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
      echo "Pipeline failed. Printing diagnostics…"
      sh '''set -eu pipefail

        echo "----- Deploy diagnostics -----"

        kubectl get deploy -n ${K8S_NAMESPACE} -l app=${APP_NAME} -o wide || true
        kubectl describe deploy/${APP_NAME}-${TARGET_COLOR} -n ${K8S_NAMESPACE} || true

        POD="$(kubectl get pod -n ${K8S_NAMESPACE} -l app=${APP_NAME},version=${TARGET_COLOR} -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)"

        if [ -n "${POD:-}" ]; then
          kubectl describe pod/${POD} -n ${K8S_NAMESPACE} || true
          kubectl logs ${POD} -n ${K8S_NAMESPACE} --tail=200 || true
        fi
          echo "----- Recent events -----"
          kubectl get events -n ${K8S_NAMESPACE} --sort-by=.lastTimestamp | tail -n 100 || true
          echo "----- Ingress annotation (weights) -----"
          kubectl get ing/${APP_NAME}-ingress -n ${K8S_NAMESPACE} -o jsonpath="{.metadata.annotations.alb\\.ingress\\.kubernetes\\.io/actions\\.forward-blue-green}" || true
          echo
          echo "Note: Traffic weights are commit-driven via traffic-patch.yaml. Revert in Git to roll back weights."
      '''
    }
    success {
      echo "Kustomize-based blue/green rollout complete. Target color: ${TARGET_COLOR}, Image: ${IMAGE_TAG}.."
    }
  }
}
