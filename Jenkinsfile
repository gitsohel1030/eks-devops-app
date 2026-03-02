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
          def release = readYaml file: 'k8s/overlays/prod/release.yaml'
          def active = release.activeColor
    
          if (!active) {
            error "activeColor not found in release.yaml"
          }
    
          env.CURRENT_COLOR = active
          env.TARGET_COLOR  = (active == "blue") ? "green" : "blue"
    
          echo "Active Color: ${env.CURRENT_COLOR}"
          echo "Target Color: ${env.TARGET_COLOR}"
        }
      }
    }   
 

    stage('Build & Apply Kustomize Overlay') {
      steps {
        script {

          echo "Using CURRENT_COLOR=${env.CURRENT_COLOR}"
          echo "Using TARGET_COLOR=${env.TARGET_COLOR}"
          echo "Using IMAGE_TAG=${env.IMAGE_TAG}"

          def OUT = "k8s/.out/prod"

          // --- 1. Clean output dir ---
          sh "rm -rf ${OUT}"
          sh "mkdir -p ${OUT}"

          // --- 2. Copy overlay ---
          sh "cp -R k8s/overlays/prod/* ${OUT}/"

          // --- 3. Choose correct image patch ---
          def patchFile = "${OUT}/patch-blue-image.yaml"

          if (env.TARGET_COLOR == "green") {
            sh """
              sed -i 's|patch-blue-image.yaml|patch-green-image.yaml|' ${OUT}/kustomization.yaml
            """
            patchFile = "${OUT}/patch-green-image.yaml"
          }

          echo "Patch selected: ${patchFile}"

          // --- 4. Stamp the image tag ---
          sh """
            sed -i 's|__IMAGE_TAG__|${env.IMAGE_TAG}|' ${patchFile}
          """

          // --- 5. Preview ---
          // echo "=== Kustomize Build Preview ==="
          // sh "kubectl kustomize ${OUT} | head -n 200"

          // --- 6. Diff vs cluster ---
          // echo "=== Diff ==="
          // sh "kubectl diff -k ${OUT} || true"

          // --- 7. Apply ---
          echo "=== Applying overlay ==="
          sh "kubectl apply -k ${OUT}"

          // --- 8. Wait for rollout ---
          def deployName = "${env.APP_NAME}-${env.TARGET_COLOR}"
          echo "=== Waiting for rollout: ${deployName} ==="

          sh """
            kubectl rollout status deploy/${deployName} -n ${env.K8S_NAMESPACE} --timeout=3m
          """

          // --- 9. Annotate ---
          sh """
            kubectl annotate deploy/${deployName} \
              -n ${env.K8S_NAMESPACE} \
              kubernetes.io/change-cause="Deploy ${env.IMAGE_TAG} to ${env.TARGET_COLOR}" \
              --overwrite || true
          """

          // --- 10. Pods output ---
          echo "=== Pods After Deploy ==="
          sh """
            kubectl get pods -n ${env.K8S_NAMESPACE} -l app=${env.APP_NAME} -o wide
          """
        }
      }
    }

    stage('Show Traffic Weights (from Git overlay)') {
      steps {
        script {
          def ing = "${env.APP_NAME}-ingress"

          def ann = sh(
            script: """
              kubectl get ingress ${ing} -n ${env.K8S_NAMESPACE} \
              -o jsonpath='{.metadata.annotations.alb\\.ingress\\.kubernetes\\.io/actions\\.forward-blue-green}' 2>/dev/null || true
            """,
            returnStdout: true
          ).trim()

          echo "Current ALB forward weights (live): ${ann ?: 'NOT SET'}"
        }
      }
    }


    
    stage('Optional: Scale down OLD color') {
      when { expression { return env.SCALE_DOWN_OLD == 'true' } }
      steps {
        script {

          def oldColor = (env.TARGET_COLOR == "green") ? "blue" : "green"
          def deploy = "${env.APP_NAME}-${oldColor}"

          echo "Scaling down OLD color deployment: ${deploy}"

          sh """
            kubectl scale deployment/${deploy} -n ${env.K8S_NAMESPACE} --replicas=0
          """
        }
      }
    }

  }

  post {
    failure {
      script {
        echo "======== PIPELINE FAILED — DEBUGGING ========"

        // 1. Deployments
        sh """
          kubectl get deploy -n ${env.K8S_NAMESPACE} -l app=${env.APP_NAME} -o wide || true
        """

        def targetDeploy = "${env.APP_NAME}-${env.TARGET_COLOR}"

        // 2. Describe target deployment
        sh """
          kubectl describe deploy/${targetDeploy} -n ${env.K8S_NAMESPACE} || true
        """

        // 3. Find pod
        def pod = sh(
          script: """
            kubectl get pod -n ${env.K8S_NAMESPACE} \
            -l app=${env.APP_NAME},version=${env.TARGET_COLOR} \
            -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true
          """,
          returnStdout: true
        ).trim()

        if (pod) {
          echo "Inspecting pod: ${pod}"

          sh "kubectl describe pod/${pod} -n ${env.K8S_NAMESPACE} || true"
          sh "kubectl logs ${pod} -n ${env.K8S_NAMESPACE} --tail=200 || true"
        } else {
          echo "No pod found for TARGET_COLOR ${env.TARGET_COLOR}"
        }

        // 4. Events
        sh """
          echo "----- Recent Events -----"
          kubectl get events -n ${env.K8S_NAMESPACE} --sort-by=.lastTimestamp | tail -n 100 || true
        """

        // 5. Ingress weights
        sh """
          echo "----- Ingress Weights -----"
          kubectl get ing/${env.APP_NAME}-ingress -n ${env.K8S_NAMESPACE} \
          -o jsonpath='{.metadata.annotations.alb\\.ingress\\.kubernetes\\.io/actions\\.forward-blue-green}' || true
        """
      }
    }

    success {
      echo "Kustomize blue/green rollout SUCCESS — Target color: ${env.TARGET_COLOR}, Image: ${env.IMAGE_TAG}"
    }
  }
}
