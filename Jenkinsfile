pipeline {
  agent any

  parameters {
    booleanParam(name: 'PROMOTE', defaultValue: false, description: 'Promote TARGET color to baseline?')
    booleanParam(name: 'ROLLBACK', defaultValue: false, description: 'Rollback to CURRENT color?')
  }

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

    stage('Set IMAGE_TAG from commit') {
      steps {
        script {
          env.IMAGE_TAG = env.GIT_COMMIT.take(7)
          echo "Using IMAGE_TAG = ${IMAGE_TAG}"
        }
      }
    }
    
    stage('Build & Push Image to ECR') {
      when { 
        expression { params.PROMOTE == false && params.ROLLBACK == false } 
      }
      steps {
        script {
          // Check if tag exists
          def exists = sh(
            script: """
              aws ecr describe-images \
                --region ${AWS_REGION} \
                --repository-name ${REPO_NAME} \
                --image-ids imageTag=${IMAGE_TAG} >/dev/null 2>&1 && echo true || echo false
            """,
            returnStdout: true
          ).trim()

          if (exists == "true") {
            echo "Image ${IMAGE_TAG} already exists — skipping build"
          } else {
            sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."

            sh """
              aws ecr get-login-password --region ${AWS_REGION} \
              | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
              docker push ${ECR_REPO}:${IMAGE_TAG}
            """
          }
        }
      }
    }


    stage('Configure Kubectl') {
      steps { 
        sh "mv ~/.kube/config ~/.kube/config.bak.\$(date +%s) 2>/dev/null || true"
        sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}" 
        }
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
      when { 
        expression { params.PROMOTE == false && params.ROLLBACK == false } 
      }
      steps {
        sh '''
          set -eu pipefail
          kubectl get ns ${K8S_NAMESPACE} >/dev/null 2>&1 || kubectl create ns ${K8S_NAMESPACE}
        '''
      }
    }

    // stage('Clone GitOps Repo') {
    //   steps {
    //     script {

    //       echo "Preparing to clone GitOps repo..."

    //       withCredentials([sshUserPrivateKey(credentialsId: 'github-ssh-gitops', keyFileVariable: 'SSH_KEY')]) {

    //         //
    //         // 1. Write SSH key to a local file
    //         //
    //         def keyText = readFile(SSH_KEY)
    //         writeFile file: 'gitops_key', text: keyText
    //         sh "chmod 600 gitops_key"

    //         //
    //         // 2. Ensure known_hosts exists
    //         //
    //         sh """
    //           mkdir -p ~/.ssh
    //           ssh-keyscan github.com >> ~/.ssh/known_hosts
    //         """

    //         //
    //         // 3. Fresh clone using explicit SSH key (NO ssh-agent needed)
    //         //
    //         sh """
    //           rm -rf ${GITOPS_DIR}
    //           GIT_SSH_COMMAND='ssh -i gitops_key -o StrictHostKeyChecking=no' \
    //           git clone ${GITOPS_REPO} ${GITOPS_DIR}
    //           """
    //       }
    //     }
    //   }
    // }


    stage('Determine TARGET color') {
      when { 
        expression { params.PROMOTE == false && params.ROLLBACK == false } 
      }
      steps {
        script {
          def release = readYaml file: 'eks-devops-gitops/k8s/overlays/prod/release.yaml'
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
 
    stage('Push changes in git') {      
      when { 
        expression { params.PROMOTE == false && params.ROLLBACK == false } 
      }
      steps {
        script {
          echo "Preparing to update GitOps repo..."
          withCredentials([sshUserPrivateKey(credentialsId: 'github-ssh-gitops', keyFileVariable: 'SSH_KEY')]) {
            withCredentials([file(credentialsId: 'github-known-hosts', variable: 'KNOWN_HOSTS')]) {
              // 1. Write SSH key to a local file
              // def keyText = readFile(SSH_KEY)
              // writeFile file: 'gitops_key', text: keyText
              // sh "chmod 600 gitops_key"

              // 2. Ensure known_hosts exists
              sh """
                mkdir -p ~/.ssh
                ssh-keyscan github.com >> ~/.ssh/known_hosts
              """

              // 3. Fresh clone using explicit SSH key (NO ssh-agent needed)
              sh """
                rm -rf ${GITOPS_DIR}
                GIT_SSH_COMMAND='ssh -i ${SSH_KEY} -o UserKnownHostsFile=${KNOWN_HOSTS} -o StrictHostKeyChecking=yes' \
                git clone ${GITOPS_REPO} ${GITOPS_DIR}
              """

              // 4. Enter GitOps repo
              dir("${GITOPS_DIR}") {
                // Ensure remote uses SSH
                sh "git remote set-url origin ${GITOPS_REPO}"

                // Checkout & pull main (using same SSH key)
                sh """
                  GIT_SSH_COMMAND='ssh -i ${SSH_KEY} -o UserKnownHostsFile=${KNOWN_HOSTS} -o StrictHostKeyChecking=yes' \
                  git checkout main
                  git pull origin main || true
                """
                
                // Update release.yaml
                //
                def relFile = "k8s/overlays/prod/release.yaml"
                def relContent = readFile(relFile)
                relContent = relContent.replaceAll(/activeColor:.*/, "activeColor: ${TARGET_COLOR}")
                writeFile file: relFile, text: relContent

                // ---- Update image patch for TARGET color only ----
                def patchFile = "k8s/overlays/prod/patch-${env.TARGET_COLOR}-image.yaml"

                def patchContent = readFile(patchFile)
                patchContent = patchContent.replaceAll(/image:.*/, "image: ${ECR_REPO}:${IMAGE_TAG}")
                writeFile file: patchFile, text: patchContent

                echo "Updated image patch → ${patchFile}"

                // ---- UPDATE TRAFFIC PATCH ----
                def trafficPatch = (env.TARGET_COLOR == "green")
                    ? "traffic/traffic-green-100.yaml"
                    : "traffic/traffic-blue-100.yaml"

                sh """
                  sed -i 's|traffic/traffic-blue-100.yaml|${trafficPatch}|g' k8s/overlays/prod/kustomization.yaml
                  sed -i 's|traffic/traffic-green-100.yaml|${trafficPatch}|g' k8s/overlays/prod/kustomization.yaml
                """
                echo "Updated traffic patch → ${trafficPatch}"

                // Git config
                sh "git config user.email '${GIT_EMAIL}'"
                sh "git config user.name '${GIT_USER}'"

                // Commit & push (explicit SSH key)
                sh """
                  git add .
                  git commit -m 'GitOps deploy ${IMAGE_TAG} to ${TARGET_COLOR}' || true
                  GIT_SSH_COMMAND='ssh -i ${SSH_KEY} -o UserKnownHostsFile=${KNOWN_HOSTS} -o StrictHostKeyChecking=yes' \
                  git push origin main
                """
              }
            }
          }
        }
      }
    }
  
    stage('Promote TARGET Version to Baseline (Manual Trigger)') {
      when {
        expression { return params.PROMOTE == true }
      }
      steps {                      
      echo "PROMOTION TRIGGERED: Syncing baseline image..."
        script {
          withCredentials([sshUserPrivateKey(credentialsId: 'github-ssh-gitops', keyFileVariable: 'SSH_KEY')]) {
            // 1. Write SSH key to a local file
            def keyText = readFile(SSH_KEY)
            writeFile file: 'gitops_key', text: keyText
            sh "chmod 600 gitops_key"

            dir("${GITOPS_DIR}") {
              // Ensure remote uses SSH
              sh "git remote set-url origin ${GITOPS_REPO}"

              // Checkout & pull main (using same SSH key)
              sh """
                GIT_SSH_COMMAND='ssh -i ../gitops_key -o StrictHostKeyChecking=no' \
                git checkout main
                git pull origin main || true
              """

              def otherColor = (env.TARGET_COLOR == "green") ? "blue" : "green"
              def targetPatch = "k8s/overlays/prod/patch-${env.TARGET_COLOR}-image.yaml"
              def otherPatch  = "k8s/overlays/prod/patch-${otherColor}-image.yaml"

              echo "TARGET patch file = ${targetPatch}"
              echo "OTHER  patch file = ${otherPatch}"

              // Extract image tag from target patch
              def imageLine = sh(script: "grep 'image:' ${GITOPS_DIR}/${targetPatch}", returnStdout: true).trim()
              def newImage = imageLine.split('image:')[1].trim()
              echo "Promoting image: ${newImage}"

              // Replace image in other patch
              def otherContent = readFile("${GITOPS_DIR}/${otherPatch}")
              otherContent = otherContent.replaceAll(/image:.*/, "image: ${newImage}")
              writeFile file: "${GITOPS_DIR}/${otherPatch}", text: otherContent

              // Commit + Push            
              dir("${GITOPS_DIR}") {
                sh "git config user.email '${GIT_EMAIL}'"
                sh "git config user.name '${GIT_USER}'"

                sh "git add ${otherFilePath}"

                sh """
                  git commit -m 'Promote ${newImage} to baseline (${otherColor})' \
                    || echo 'No promotion commit needed'
                """

                sh """
                  GIT_SSH_COMMAND='ssh -i ../gitops_key -o StrictHostKeyChecking=no' \
                  git push origin main
                """

                echo "🎉 PROMOTION COMPLETE: Baseline updated to ${newImage}"
              }
            }
          }
        }
      }
    } 


    
    stage('Rollback Last Deployment (Manual)') {
      when { expression { params.ROLLBACK == true } }
      steps {
        script {

          withCredentials([sshUserPrivateKey(credentialsId: 'github-ssh-gitops', keyFileVariable: 'SSH_KEY')]) {

            writeFile file: 'gitops_key', text: readFile(SSH_KEY)
            sh "chmod 600 gitops_key"

            // Revert last commit
            sh """
              GIT_SSH_COMMAND='ssh -i gitops_key -o StrictHostKeyChecking=no' \
              git -C ${GITOPS_DIR} revert --no-edit HEAD
            """

            sh """
              GIT_SSH_COMMAND='ssh -i gitops_key -o StrictHostKeyChecking=no' \
              git -C ${GITOPS_DIR} push origin main
            """
          }
        }
      }
    }

    // -------------------------------------------------------------
    // 7. ArgoCD Notes (Informational)
    // WHY: Deployment is now fully GitOps-driven....
    // -------------------------------------------------------------
    stage('GitOps Info') {
      steps {
        echo """
        GitOps Update Complete!
        ArgoCD is now responsible for syncing these changes to the cluster.

        Deployment Flow:
        Jenkins → Git Commit → ArgoCD Auto-Sync → Kubernetes

        New ACTIVE COLOR will be after ArgoCD sync: ${env.TARGET_COLOR}
        """
      }
    }
  }

  post {
    success {
      echo "GitOps Blue/Green update complete — Target Color: ${env.TARGET_COLOR}"
    }
    failure {
      echo "Something went wrong during GitOps update."
    }
  }  
}
