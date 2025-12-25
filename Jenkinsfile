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
        sh '''
          echo "📍 Current K8s context:"
          kubectl config current-context
          kubectl cluster-info || true
          echo "✅ Checkout complete"
        '''
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
          } else {
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
          sh """
            kubectl get namespace ${params.ENV} >/dev/null 2>&1 || kubectl create namespace ${params.ENV}
            kubectl get namespace ${params.ENV}
            echo "✅ Namespace ${params.ENV} ready!"
          """
        }
      }
    }

    stage('Apply Storage') {
      steps {
        timeout(time: 12, unit: 'MINUTES') {
          sh """
            ENV_NS="${params.ENV}"
            PV_NAME="shared-pv-\$ENV_NS"
            PVC_NAME="shared-pvc"
            
            echo "🔧 Fixing storage for \$ENV_NS (minikube)..."
            
            # Aggressive cleanup
            echo "🧹 Force deleting existing resources..."
            kubectl delete pvc \$PVC_NAME -n \$ENV_NS --ignore-not-found --force --grace-period=0 || true
            kubectl delete pv \$PV_NAME --ignore-not-found --force --grace-period=0 || true
            sleep 10
            
            # Apply in correct order
            echo "📦 Applying StorageClass..."
            kubectl apply -f k8s/shared-storage-class.yaml || { echo "❌ StorageClass failed"; exit 1; }
            
            echo "📦 Applying PV..."
            kubectl apply -f k8s/shared-pv_\${ENV_NS}.yaml || { echo "❌ PV failed"; exit 1; }
            
            # Wait for PV to be available
            for i in {1..12}; do
              PV_STATUS=\$(kubectl get pv \$PV_NAME -o jsonpath='{.status.phase}' 2>/dev/null || echo "None")
              echo "PV [\$i/12]: \$PV_STATUS"
              if [ "\$PV_STATUS" = "Available" ]; then
                echo "✅ PV Available!"
                break
              fi
              sleep 10
            done
            
            echo "📦 Applying PVC..."
            kubectl apply -f k8s/shared-pvc_\${ENV_NS}.yaml -n \$ENV_NS || { echo "❌ PVC failed"; exit 1; }
            
            # Wait for PVC to bind (minikube friendly)
            echo "⏳ Waiting PVC to bind (max 120s)..."
            for i in {1..12}; do
              PVC_STATUS=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
              PVC_BOUND=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.status.conditions[?(@.type=="FileSystemResizePending")].status}' 2>/dev/null || echo "None")
              echo "[\$i/12] PVC: \$PVC_STATUS | Bound: \$PVC_BOUND"
              
              if [ "\$PVC_STATUS" = "Bound" ]; then 
                echo "🎉 PVC BOUND SUCCESS!"
                break 
              fi
              sleep 10
            done
            
            # Final verification
            kubectl get pv \$PV_NAME -o wide
            kubectl get pvc \$PVC_NAME -n \$ENV_NS -o wide
            kubectl describe pvc \$PVC_NAME -n \$ENV_NS | grep -A5 -B5 "Events"
            echo "✅ STORAGE READY FOR MINIKUBE!"
          """
        }
      }
    }

    stage('Verify Docker Secret') {
      steps {
        timeout(time: 30, unit: 'SECONDS') {
          sh "kubectl get secret regcred -n ${params.ENV} >/dev/null 2>&1 && echo '✅ Secret OK' || echo '⚠️ Continuing...'"
        }
      }
    }

    stage('Docker Login') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
          sh "echo \"\$HARBOR_PASS\" | docker login ${REGISTRY} -u \"\$HARBOR_USER\" --password-stdin && echo '✅ Docker login OK'"
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
        withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
          sh """
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${params.ENV}.yaml
            git config user.name "Thanuja"
            git config user.email "ratakondathanuja@gmail.com"
            git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml version.txt || true
            git commit -m "chore: images ${IMAGE_TAG} for ${params.ENV} [skip ci]" || true
            git push https://\$GIT_USER:\$GIT_TOKEN@github.com/ThanujaRatakonda/kp_10.git master || true
            echo "✅ Git updated"
          """
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        timeout(time: 3, unit: 'MINUTES') {
          sh """
            kubectl apply -f argocd/backend_${params.ENV}.yaml || true
            kubectl apply -f argocd/frontend_${params.ENV}.yaml || true
            kubectl apply -f argocd/database-app_${params.ENV}.yaml || true
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
            kubectl get pods -n ${params.ENV} -o wide || true
            kubectl get svc -n ${params.ENV} || true
            kubectl get pvc -n ${params.ENV} || true
            kubectl get applications -n argocd | grep -E "(frontend|backend|database)" || true
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
      sh """
        echo "=== DEBUG INFO ==="
        kubectl get pvc -n ${params.ENV} -o yaml || true
        kubectl get events -n ${params.ENV} --sort-by=.lastTimestamp | tail -10 || true
      """ || true
    }
  }
}
