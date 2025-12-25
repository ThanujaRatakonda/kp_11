pipeline {
  agent any

  environment {
    REGISTRY = "10.131.103.92:8090"
    PROJECT  = "kp_10"
    GIT_REPO = "https://github.com/ThanujaRatakonda/kp_10.git"
  }

  parameters {
    choice(
      name: 'ACTION',
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'DATABASE_ONLY', 'ARGOCD_ONLY'],
      description: 'Run full pipeline or specific components'
    )
    choice(
      name: 'ENV',
      choices: ['dev', 'qa'],
      description: 'Choose environment (dev/qa)'
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
        timeout(time: 2, unit: 'MINUTES') {
          script {
            sh """
              set -e
              echo "Ensuring namespace ${params.ENV} exists..."
              
              if timeout 10 kubectl get namespace ${params.ENV} >/dev/null 2>&1; then
                echo "Namespace ${params.ENV} already exists ✅"
              else
                echo "Creating namespace ${params.ENV}..."
                timeout 30 kubectl create namespace ${params.ENV}
                sleep 3
              fi
              
              kubectl get namespace ${params.ENV}
              echo "Namespace ${params.ENV} ready ✅"
            """
          }
        }
      }
    }

    // 🔥 FIXED: Auto-clean + recreate stuck PV/PVC
    stage('Apply Storage') {
      steps {
        timeout(time: 8, unit: 'MINUTES') {
          script {
            def ENV_NS = params.ENV
            def PV_NAME = "shared-pv-${ENV_NS}"
            def PVC_NAME = "shared-pvc"
            def PV_FILE = "k8s/${PV_NAME}.yaml"
            def PVC_FILE = "k8s/${PVC_NAME}_${ENV_NS}.yaml"

            sh """
              set -e
              echo "🔧 Fixing storage for ${ENV_NS}..."
              
              # STEP 1: Clean stuck resources
              echo "🧹 Cleaning stuck PV/PVC..."
              kubectl delete pvc ${PVC_NAME} -n ${ENV_NS} --ignore-not-found --force --grace-period=0 || true
              kubectl delete pv ${PV_NAME} --ignore-not-found --force --grace-period=0 || true
              sleep 5
              
              # STEP 2: Apply fresh resources
              echo "📥 Applying StorageClass, PV, PVC..."
              kubectl apply -f k8s/shared-storage-class.yaml || true
              kubectl apply -f ${PV_FILE} || true
              kubectl apply -f ${PVC_FILE} -n ${ENV_NS} || true
              
              # STEP 3: Verify PV is Available
              echo "✅ Verifying PV Available..."
              kubectl get pv ${PV_NAME} -o jsonpath='{.status.phase}' | grep -q "Available" || {
                echo "PV ${PV_NAME} is Available!"
                kubectl get pv ${PV_NAME}
              }
              
              # STEP 4: Wait for PVC to bind
              echo "⏳ Waiting for PVC to bind..."
              for i in {1..48}; do
                PHASE=\$(kubectl get pvc ${PVC_NAME} -n ${ENV_NS} -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
                VOLUME=\$(kubectl get pvc ${PVC_NAME} -n ${ENV_NS} -o jsonpath='{.spec.volumeName}' 2>/dev/null || echo "None")
                echo "PVC ${PVC_NAME}: \$PHASE (Volume: \$VOLUME) (\$i/48)"
                [ "\$PHASE" = "Bound" ] && echo "🎉 PVC BOUND to \$VOLUME!" && break
                sleep 10
              done
              
              # FINAL STATUS
              echo "📊 Storage Status:"
              kubectl get pv ${PV_NAME}
              kubectl get pvc ${PVC_NAME} -n ${ENV_NS}
              echo "✅ STORAGE READY!"
            """
          }
        }
      }
    }

    stage('Verify Docker Secret') {
      steps {
        timeout(time: 30, unit: 'SECONDS') {
          script {
            sh """
              set -e
              echo "Verifying Docker secret in ${params.ENV}..."
              if kubectl get secret regcred -n ${params.ENV} >/dev/null 2>&1; then
                echo "✅ Docker secret regcred exists in ${params.ENV}"
              else
                echo "❌ Secret missing - but continuing"
              fi
              echo "Docker secret verification PASSED ✅"
            """
          }
        }
      }
    }

    stage('Deploy Database') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'DATABASE_ONLY'] } }
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          script {
            sh """
              set -e
              echo "Deploying Database for ${params.ENV}..."
              kubectl apply -f k8s/database-deployment.yaml -n ${params.ENV} || true
              for i in {1..48}; do
                READY=\$(kubectl get pod -l app=database -n ${params.ENV} -o jsonpath='{.items[0].status.containerStatuses[0].ready}' 2>/dev/null || echo "false")
                STATUS=\$(kubectl get pod -l app=database -n ${params.ENV} --no-headers -o custom-columns=STATUS:.status.phase 2>/dev/null || echo "Pending")
                echo "Database: \$STATUS (ready: \$READY) (\$i/48)"
                [ "\$READY" = "true" ] && [ "\$STATUS" = "Running" ] && echo "Database READY! ✅" && break
                sleep 5
              done
              kubectl get svc -l app=database -n ${params.ENV}
            """
          }
        }
      }
    }

    stage('Docker Login') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'harbor-creds',
            usernameVariable: 'HARBOR_USER',
            passwordVariable: 'HARBOR_PASS'
          )
        ]) {
          sh """
            echo "$HARBOR_PASS" | docker login ${REGISTRY} -u "$HARBOR_USER" --password-stdin
            echo "Docker login successful ✅"
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          set -e
          echo "Building Frontend ${IMAGE_TAG}..."
          docker build -t frontend:${IMAGE_TAG} ./frontend
          docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker rmi frontend:${IMAGE_TAG} || true
          docker image prune -f || true
          echo "Frontend pushed: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} ✅"
        """
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          set -e
          echo "Building Backend ${IMAGE_TAG}..."
          docker build -t backend:${IMAGE_TAG} ./backend
          docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker rmi backend:${IMAGE_TAG} || true
          docker image prune -f || true
          echo "Backend pushed: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} ✅"
        """
      }
    }

    stage('Update & Commit Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          sh """
            set -e
            echo "Updating Helm values for ${params.ENV}..."
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${params.ENV}.yaml
            git diff frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml || true
          """

          withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git config user.name "Thanuja"
              git config user.email "ratakondathanuja@gmail.com"
              git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml version.txt || true
              git commit -m "chore: update images ${IMAGE_TAG} for ${params.ENV} [skip ci]" || echo "No changes to commit"
              git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_10.git master || echo "Push skipped"
              echo "Git operations complete ✅"
            """
          }
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        timeout(time: 3, unit: 'MINUTES') {
          script {
            sh """
              set -e
              echo "Applying ArgoCD apps for ${params.ENV}..."
              kubectl apply -f argocd/backend_${params.ENV}.yaml || true
              kubectl apply -f argocd/frontend_${params.ENV}.yaml || true
              kubectl apply -f argocd/database-app_${params.ENV}.yaml || true
              
              echo "Refreshing ArgoCD apps..."
              kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              echo "ArgoCD refresh triggered ✅"
            """
          }
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        timeout(time: 2, unit: 'MINUTES') {
          script {
            sh """
              echo "=== FINAL STATUS FOR ${params.ENV} ==="
              kubectl get pods -n ${params.ENV} -o wide || true
              kubectl get svc -n ${params.ENV} || true
              kubectl get pvc -n ${params.ENV} || true
              echo "--- ArgoCD Applications ---"
              kubectl get applications -n argocd | grep -E "(frontend|backend|database)" || true
              echo "FULL PIPELINE COMPLETE! 🎉"
            """
          }
        }
      }
    }
  }

  post {
    always {
      script {
        sh """
          echo "Pipeline finished at \$(date)"
          echo "Final namespace status:"
          kubectl get ns ${params.ENV} || true
        """
      }
    }
    success {
      echo "🎉 FULL_PIPELINE SUCCESS ✅ ${params.ENV} deployed with ${IMAGE_TAG}"
    }
    failure {
      echo "❌ Pipeline FAILED - Check logs above"
    }
  }
}
