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
    
    GITOPS_REPO     = "git@github.com:gitsohel1030/eks-devops-gitops.git"
    GITOPS_DIR      = "eks-devops-gitops"
    GIT_USER        = "Sohel"
    GIT_EMAIL       = "sohelmujawar172@gmail.com"

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
 

    // -------------------------------------------------------------
    // 5. Update Git Manifests for GitOps Sync
    // WHY: Git is the ONLY source of truth in GitOps.
    // WHAT: Update release.yaml + patch-<color>-image.yaml
    // -------------------------------------------------------------
    
    stage('GitOps Update') {
      steps {
        script {

          echo "Preparing to update GitOps repo..."

          withCredentials([sshUserPrivateKey(credentialsId: 'github-ssh-gitops', keyFileVariable: 'SSH_KEY')]) {

            //
            // 1. Write SSH key to a local file
            //
            def keyText = readFile(SSH_KEY)
            writeFile file: 'gitops_key', text: keyText
            sh "chmod 600 gitops_key"

            //
            // 2. Ensure known_hosts exists
            //
            sh """
              mkdir -p ~/.ssh
              ssh-keyscan github.com >> ~/.ssh/known_hosts
            """

            //
            // 3. Fresh clone using explicit SSH key (NO ssh-agent needed)
            //
            sh """
              rm -rf ${GITOPS_DIR}
              GIT_SSH_COMMAND='ssh -i gitops_key -o StrictHostKeyChecking=no' \
              git clone ${GITOPS_REPO} ${GITOPS_DIR}
            """

            //
            // 4. Enter GitOps repo
            //
            dir("${GITOPS_DIR}") {

              //
              // Ensure remote uses SSH
              //
              sh "git remote set-url origin ${GITOPS_REPO}"

              //
              // Checkout & pull main (using same SSH key)
              //
              sh """
                GIT_SSH_COMMAND='ssh -i ../gitops_key -o StrictHostKeyChecking=no' \
                git checkout main
              """

              sh """
                GIT_SSH_COMMAND='ssh -i ../gitops_key -o StrictHostKeyChecking=no' \
                git pull origin main || true
              """

              //
              // Update release.yaml
              //
              def relFile = "k8s/overlays/prod/release.yaml"
              def relContent = readFile(relFile)
              relContent = relContent.replaceAll(/activeColor:.*/, "activeColor: ${TARGET_COLOR}")
              writeFile file: relFile, text: relContent

              //
              // Update patch-<color>-image.yaml
              //
              def patchFile = "k8s/overlays/prod/patch-${TARGET_COLOR}-image.yaml"
              def patchContent = readFile(patchFile)
              patchContent = patchContent.replaceAll(/__IMAGE_TAG__/, IMAGE_TAG)
              writeFile file: patchFile, text: patchContent

              //
              // Git config
              //
              sh "git config user.email '${GIT_EMAIL}'"
              sh "git config user.name '${GIT_USER}'"

              //
              // Commit & push (explicit SSH key)
              //
              sh """
                git add .
                git commit -m 'GitOps deploy ${IMAGE_TAG} to ${TARGET_COLOR}' || true
                GIT_SSH_COMMAND='ssh -i ../gitops_key -o StrictHostKeyChecking=no' \
                git push origin main
              """
            }
          }
        }
      }
    }
  


    // -------------------------------------------------------------
    // 7. ArgoCD Notes (Informational)
    // WHY: Deployment is now fully GitOps-driven...
    // -------------------------------------------------------------
    stage('GitOps Info') {
      steps {
        echo """
        🚀 GitOps Update Complete!
        ArgoCD is now responsible for syncing these changes to the cluster.

        Deployment Flow:
        Jenkins → Git Commit → ArgoCD Auto-Sync → Kubernetes

        New ACTIVE COLOR will be after ArgoCD sync: ${env.TARGET_COLOR}
        """
      }
    }
  }

  // -------------------------------------------------------------
  // POST STEPS
  // -------------------------------------------------------------
  post {
    success {
      echo "GitOps Blue/Green update complete — Target Color: ${env.TARGET_COLOR}"
    }
    failure {
      echo "❌ Something went wrong during GitOps update."
    }
  }
}
