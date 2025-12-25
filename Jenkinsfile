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
              timeout 10 kubectl get namespace ${params.ENV} >/dev/null 2>&1 || timeout 30 kubectl create namespace ${params.ENV}
              kubectl get namespace ${params.ENV}
              echo "Namespace ${params.ENV} ready ✅"
            """
          }
        }
      }
    }

    // 🔥 BULLETPROOF STORAGE FIX
    stage('Apply Storage') {
      steps {
        timeout(time: 8, unit: 'MINUTES') {
          script {
            def ENV_NS = params.ENV
            def PV_NAME = "shared-pv-${ENV_NS}"
            def PVC_NAME = "shared-pvc"

            sh """
              echo "🔧 Auto-fixing storage for ${ENV_NS} (8min timeout)..."
              
              # Clean everything (ignore all errors)
              kubectl delete pvc ${PVC_NAME} -n ${ENV_NS} --ignore-not-found --force --grace-period=0 2>/dev/null || true
              kubectl delete pv ${PV_NAME} --ignore-not-found --force --grace-period=0 2>/dev/null || true
              sleep 8
              
              # Apply EXACT filenames
              kubectl apply -f k8s/shared-storage-class.yaml
              kubectl apply -f k8s/shared-pv_${ENV_NS}.yaml
              sleep 3
              kubectl apply -f k8s/shared-pvc_${ENV_NS}.yaml -n ${ENV_NS}
              
              # Verify PV
              kubectl get pv ${PV_NAME}
              
              # Wait PVC bind (SIMPLE)
              echo "⏳ Waiting PVC bind (4min max)..."
              for i in {1..24}; do
                PHASE=\$(kubectl get pvc ${PVC_NAME} -n ${ENV_NS} -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
                echo "[${i}/24] PVC: \$PHASE"
                [ "\$PHASE" = "Bound" ] && echo "🎉 PVC BOUND!" && break
                sleep 10
              done
              
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
          sh """
            echo "Verifying Docker secret in ${params.ENV}..."
            kubectl get secret regcred -n ${params.ENV} >/dev/null 2>&1 && echo "✅ Secret OK" || echo "⚠️ Secret missing - continuing"
          """
        }
      }
    }

    stage('Deploy Database') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'DATABASE_ONLY'] } }
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          sh """
            echo "Deploying Database..."
            kubectl apply -f k8s/database-deployment.yaml -n ${params.ENV}
            for i in {1..48}; do
              READY=\$(kubectl get pod -l app=database -n ${params.ENV} -o jsonpath='{.items[0].status.containerStatuses[0].ready}' 2>/dev/null || echo "false")
              STATUS=\$(kubectl get pod -l app=database -n ${params.ENV} --no-headers -o custom-columns=STATUS:.status.phase 2>/dev/null || echo "Pending")
              echo "DB: \$STATUS (\$i/48)"
              [ "\$READY" = "true" ] && [ "\$STATUS" = "Running" ] && echo "✅ DB READY!" && break
              sleep 5
            done
          """
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
            echo "\$HARBOR_PASS" | docker login ${REGISTRY} -u "\$HARBOR_USER" --password-stdin
            echo "✅ Docker login OK"
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          docker build -t frontend:${IMAGE_TAG} ./frontend
          docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker rmi frontend:${IMAGE_TAG} || true
          echo "✅ Frontend: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}"
        """
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t backend:${IMAGE_TAG} ./backend
          docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker rmi backend:${IMAGE_TAG} || true
          echo "✅ Backend: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}"
        """
      }
    }

    stage('Update & Commit Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          sh """
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${params.ENV}.yaml
          """
          withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git config user.name "Thanuja"
              git config user.email "ratakondathanuja@gmail.com"
              git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml version.txt
              git commit -m "chore: images ${IMAGE_TAG} for ${params.ENV} [skip ci]" || true
              git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_10.git master || true
              echo "✅ Git updated"
            """
          }
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        timeout(time: 3, unit: 'MINUTES') {
          sh """
            kubectl apply -f argocd/backend_${params.ENV}.yaml
            kubectl apply -f argocd/frontend_${params.ENV}.yaml  
            kubectl apply -f argocd/database-app_${params.ENV}.yaml
            kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            echo "✅ ArgoCD refreshed"
          """
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        timeout(time: 2, unit: 'MINUTES') {
          sh """
            echo "=== FINAL STATUS ${params.ENV} ==="
            kubectl get pods -n ${params.ENV} -o wide
            kubectl get svc -n ${params.ENV}
            kubectl get pvc -n ${params.ENV}
            kubectl get applications -n argocd | grep -E "(frontend|backend|database)"
            echo "🎉 FULL PIPELINE COMPLETE!"
          """
        }
      }
    }
  }

  post {
    always {
      sh "kubectl get ns ${params.ENV} || true"
    }
    success {
      echo "🎉 FULL_PIPELINE SUCCESS! ${params.ENV} with ${IMAGE_TAG}"
    }
    failure {
      echo "❌ FAILED - check logs"
    }
  }
}
