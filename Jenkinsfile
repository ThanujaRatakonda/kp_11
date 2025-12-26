pipeline {
  agent any

  environment {
    REGISTRY = "10.131.103.92:8090"
    PROJECT  = "kp_11"
    GIT_REPO = "https://github.com/ThanujaRatakonda/kp_11.git"
  }

  parameters {
    choice(
      name: 'ACTION',
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'DATABASE_ONLY', 'ARGOCD_ONLY'],
      description: 'Run full pipeline or specific components'
    )
    choice(
      name: 'ENV',
      choices: ['dev', 'qa', 'both'],
      description: 'Choose environment (dev/qa/both)'
    )
    choice(
      name: 'VERSION_BUMP',
      choices: ['patch', 'minor', 'major'],
      description: 'Choose version bump: patch/minor/major'
    )
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'git-creds', url: "${GIT_REPO}", branch: 'master'
      }
    }
    
    stage('Read & Update Version') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          def newVersion

          if (params.VERSION_BUMP == 'patch') {
            def patchFile = 'patch_counter.txt'
            def currentPatch = fileExists(patchFile) ? readFile(patchFile).trim().toInteger() : -1 
            newVersion = "v1.0.${currentPatch + 1}"
            writeFile file: patchFile, text: "${currentPatch + 1}"
          } else if (params.VERSION_BUMP == 'minor') {
            def minorFile = 'minor_counter.txt'
            def currentMinor = fileExists(minorFile) ? readFile(minorFile).trim().toInteger() : -1
            newVersion = "v1.1.${currentMinor + 1}"
            writeFile file: minorFile, text: "${currentMinor + 1}"
          } else if (params.VERSION_BUMP == 'major') {
            def majorFile = 'major_counter.txt'
            def currentMajor = fileExists(majorFile) ? readFile(majorFile).trim().toInteger() : -1
            newVersion = "v2.0.${currentMajor + 1}"
            writeFile file: majorFile, text: "${currentMajor + 1}"
          }

          env.IMAGE_TAG = newVersion
          writeFile file: 'version.txt', text: newVersion
          echo "${params.VERSION_BUMP} → ${newVersion}"
        }
      }
    }

    stage('Create Namespace') {
      steps {
        script {
          if (params.ENV == 'both') {
            sh """
              kubectl get namespace dev >/dev/null 2>&1 || kubectl create namespace dev
              kubectl get namespace qa >/dev/null 2>&1 || kubectl create namespace qa
              kubectl get namespace argocd >/dev/null 2>&1 || kubectl create namespace argocd
              echo "Namespaces: dev + qa + argocd"
            """
          } else {
            sh """
              kubectl get namespace ${params.ENV} >/dev/null 2>&1 || kubectl create namespace ${params.ENV}
              kubectl get namespace argocd >/dev/null 2>&1 || kubectl create namespace argocd
              echo "Namespaces: ${params.ENV} + argocd"
            """
          }
        }
      }
    }

    stage('Apply Docker Registry Secret') {
      steps {
        script {
          if (params.ENV == 'both') {
            parallel([
              'dev': { sh 'kubectl apply -f dockersecret_dev.yaml'; sh 'kubectl get secret regcred -n dev' },
              'qa':  { sh 'kubectl apply -f dockersecret_qa.yaml'; sh 'kubectl get secret regcred -n qa' }
            ])
          } else {
            if (params.ENV == 'dev') {
              sh 'kubectl apply -f dockersecret_dev.yaml'
            } else {
              sh 'kubectl apply -f dockersecret_qa.yaml'
            }
            sh "kubectl get secret regcred -n ${params.ENV}"
          }
        }
      }
    }

    stage('Apply Storage') {
      steps {
        timeout(time: 8, unit: 'MINUTES') {
          script {
            if (params.ENV == 'both') {
              parallel([
                'dev': {
                  sh """
                    ENV_NS="dev"
                    PV_NAME="shared-pv-\$ENV_NS"
                    PVC_NAME="shared-pvc"
                    PVC_STATUS=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "None")
                    if [ "\$PVC_STATUS" = "Bound" ]; then
                      echo "PVC \$PVC_NAME already BOUND in dev → SKIPPING!"
                      kubectl get pv \$PV_NAME
                      kubectl get pvc \$PVC_NAME -n dev
                      exit 0
                    fi
                    echo "PVC not bound, fixing storage for dev..."
                    kubectl patch pv \$PV_NAME -p '{"spec":{"claimRef":null}}' || true
                    kubectl delete pvc \$PVC_NAME -n dev --force --grace-period=0 || true
                    sleep 10
                    kubectl apply -f k8s/shared-storage-class.yaml || true
                    kubectl apply -f k8s/shared-pv_dev.yaml || true
                    sleep 3
                    kubectl apply -f k8s/shared-pvc_dev.yaml -n dev || true
                    for i in {1..12}; do
                      PV_STATUS=\$(kubectl get pv \$PV_NAME -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
                      echo "PV dev[\$i/12]: \$PV_STATUS"
                      [ "\$PV_STATUS" = "Available" ] && break || sleep 5
                    done
                    for i in {1..30}; do
                      PHASE=\$(kubectl get pvc \$PVC_NAME -n dev -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
                      echo "PVC dev[\$i/30]: \$PHASE"
                      [ "\$PHASE" = "Bound" ] && break || sleep 5
                    done
                    kubectl get pv \$PV_NAME
                    kubectl get pvc \$PVC_NAME -n dev
                  """
                },
                'qa': {
                  sh """
                    ENV_NS="qa"
                    PV_NAME="shared-pv-\$ENV_NS"
                    PVC_NAME="shared-pvc"
                    PVC_STATUS=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "None")
                    if [ "\$PVC_STATUS" = "Bound" ]; then
                      echo "PVC \$PVC_NAME already BOUND in qa → SKIPPING!"
                      kubectl get pv \$PV_NAME
                      kubectl get pvc \$PVC_NAME -n qa
                      exit 0
                    fi
                    echo "PVC not bound, fixing storage for qa..."
                    kubectl patch pv \$PV_NAME -p '{"spec":{"claimRef":null}}' || true
                    kubectl delete pvc \$PVC_NAME -n qa --force --grace-period=0 || true
                    sleep 10
                    kubectl apply -f k8s/shared-storage-class.yaml || true
                    kubectl apply -f k8s/shared-pv_qa.yaml || true
                    sleep 3
                    kubectl apply -f k8s/shared-pvc_qa.yaml -n qa || true
                    for i in {1..12}; do
                      PV_STATUS=\$(kubectl get pv \$PV_NAME -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
                      echo "PV qa[\$i/12]: \$PV_STATUS"
                      [ "\$PV_STATUS" = "Available" ] && break || sleep 5
                    done
                    for i in {1..30}; do
                      PHASE=\$(kubectl get pvc \$PVC_NAME -n qa -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
                      echo "PVC qa[\$i/30]: \$PHASE"
                      [ "\$PHASE" = "Bound" ] && break || sleep 5
                    done
                    kubectl get pv \$PV_NAME
                    kubectl get pvc \$PVC_NAME -n qa
                  """
                }
              ])
            } else {
              sh """
                ENV_NS="${params.ENV}"
                PV_NAME="shared-pv-\$ENV_NS"
                PVC_NAME="shared-pvc"
                
                PVC_STATUS=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "None")
                if [ "\$PVC_STATUS" = "Bound" ]; then
                  echo " PVC \$PVC_NAME already BOUND in \$ENV_NS → SKIPPING!"
                  kubectl get pv \$PV_NAME
                  kubectl get pvc \$PVC_NAME -n \$ENV_NS
                  exit 0
                fi
                
                echo "PVC not bound, fixing storage..."
                kubectl patch pv \$PV_NAME -p '{"spec":{"claimRef":null}}' || true
                kubectl delete pvc \$PVC_NAME -n \$ENV_NS --force --grace-period=0 || true
                sleep 10
                
                kubectl apply -f k8s/shared-storage-class.yaml || true
                kubectl apply -f k8s/shared-pv_\${ENV_NS}.yaml || true
                sleep 3
                kubectl apply -f k8s/shared-pvc_\${ENV_NS}.yaml -n \$ENV_NS || true
                
                for i in {1..12}; do
                  PV_STATUS=\$(kubectl get pv \$PV_NAME -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
                  echo "PV[\$i/12]: \$PV_STATUS"
                  [ "\$PV_STATUS" = "Available" ] && break || sleep 5
                done
                
                for i in {1..30}; do
                  PHASE=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
                  echo "PVC[\$i/30]: \$PHASE"
                  [ "\$PHASE" = "Bound" ] && break || sleep 5
                done
                
                kubectl get pv \$PV_NAME
                kubectl get pvc \$PVC_NAME -n \$ENV_NS
              """
            }
          }
        }
      }
    }

    stage('Docker Login') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')
        ]) {
          sh """
            echo "$HARBOR_PASS" | docker login ${REGISTRY} -u "$HARBOR_USER" --password-stdin
            echo "Docker login successful"
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          set -e
          docker build -t frontend:${IMAGE_TAG} ./frontend
          docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker rmi frontend:${IMAGE_TAG} || true
          docker image prune -f || true
          echo "Frontend: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}"
        """
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          set -e
          docker build -t backend:${IMAGE_TAG} ./backend
          docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker rmi backend:${IMAGE_TAG} || true
          docker image prune -f || true
          echo "Backend: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}"
        """
      }
    }

    stage('Update & Commit Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          if (params.ENV == 'both') {
            sh """
              set -e
              echo "Updating Helm values for both dev+qa..."
              sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_dev.yaml
              sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_dev.yaml
              sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_qa.yaml
              sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_qa.yaml
              sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_dev.yaml
              sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_dev.yaml
              sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_qa.yaml
              sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_qa.yaml
            """
          } else {
            sh """
              set -e
              echo "Updating Helm values..."
              sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${params.ENV}.yaml
              sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${params.ENV}.yaml
              sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${params.ENV}.yaml
              sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${params.ENV}.yaml
            """
          }

          withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            def commitMsg = params.ENV == 'both' ? "chore: images ${IMAGE_TAG} for dev+qa" : "chore: images ${IMAGE_TAG} for ${params.ENV}"
            sh """
              git config user.name "Thanuja"
              git config user.email "ratakondathanuja@gmail.com"
              git add frontend-hc/frontendvalues_*.yaml backend-hc/backendvalues_*.yaml version.txt || true
              git commit -m "${commitMsg}" || echo "No changes"
              git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_11.git master
            """
          }
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        script {
          if (params.ENV == 'both') {
            parallel([
              'dev': {
                sh """
                  kubectl apply -f argocd/backend_dev.yaml
                  kubectl apply -f argocd/frontend_dev.yaml
                  kubectl apply -f argocd/database-app_dev.yaml
                  kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
                  kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
                  kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
                  echo "ArgoCD dev refreshed"
                """
              },
              'qa': {
                sh """
                  kubectl apply -f argocd/backend_qa.yaml
                  kubectl apply -f argocd/frontend_qa.yaml
                  kubectl apply -f argocd/database-app_qa.yaml
                  kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
                  kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
                  kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
                  echo "ArgoCD qa refreshed"
                """
              }
            ])
          } else {
            sh """
              kubectl apply -f argocd/backend_${params.ENV}.yaml
              kubectl apply -f argocd/frontend_${params.ENV}.yaml
              kubectl apply -f argocd/database-app_${params.ENV}.yaml
              
              kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              echo "ArgoCD ${params.ENV} refreshed"
            """
          }
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        script {
          if (params.ENV == 'both') {
            sh """
              echo "=== DEV Status ==="
              kubectl get pods -n dev
              kubectl get svc -n dev
              echo "=== QA Status ==="
              kubectl get pods -n qa
              kubectl get svc -n qa
            """
          } else {
            sh """
              kubectl get pods -n ${params.ENV}
              kubectl get svc -n ${params.ENV}
            """
          }
          sh "kubectl get applications -n argocd | grep -E '(backend|frontend|database)'"
        }
      }
    }
  }
}
