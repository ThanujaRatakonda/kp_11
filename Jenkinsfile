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
        sh """
          kubectl get namespace ${params.ENV} >/dev/null 2>&1 || kubectl create namespace ${params.ENV}
          echo "Namespace ${params.ENV} ready ✅"
        """
      }
    }

    stage('Apply Storage') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          sh """
            set -e
            
            ENV_NS="${params.ENV}"
            PV_NAME="shared-pv-\$ENV_NS"
            PVC_NAME="shared-pvc"
            
            echo "🔧 Fixing PVC binding for \$ENV_NS..."
            
            # 1. FORCE DELETE everything first
            kubectl delete pvc \$PVC_NAME -n \$ENV_NS --ignore-not-found --force --grace-period=0 || true
            kubectl delete pv \$PV_NAME --ignore-not-found --force --grace-period=0 || true
            sleep 8
            
            # 2. Create StorageClass
            kubectl apply -f k8s/shared-storage-class.yaml
            
            # 3. Create PV first and wait for Available
            kubectl apply -f k8s/shared-pv_\${ENV_NS}.yaml
            echo "⏳ Waiting for PV to be Available..."
            for i in {1..12}; do
              PV_STATUS=\$(kubectl get pv \$PV_NAME -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
              echo "PV [\$i/12]: \$PV_STATUS"
              [ "\$PV_STATUS" = "Available" ] && echo "✅ PV READY!" && break
              sleep 8
            done
            
            # 4. Create PVC
            kubectl apply -f k8s/shared-pvc_\${ENV_NS}.yaml -n \$ENV_NS
            
            # 5. Wait for PVC to bind
            echo "⏳ Waiting for PVC to bind..."
            for i in {1..30}; do
              PHASE=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
              PV_NAME_BOUND=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.spec.volumeName}' 2>/dev/null || echo "None")
              echo "[\$i/30] PVC: \$PHASE | Bound to: \$PV_NAME_BOUND"
              [ "\$PHASE" = "Bound" ] && echo "🎉 PVC BOUND SUCCESS!" && break
              sleep 6
            done
            
            # 6. FINAL CHECK
            kubectl get pv \$PV_NAME -o wide
            kubectl get pvc \$PVC_NAME -n \$ENV_NS -o wide
            echo "✅ STORAGE PERFECTLY BOUND!"
          """
        }
      }
    }

    stage('Apply Docker Secret') {
      steps {
        sh """
          kubectl apply -f docker-registry-secret.yaml -n ${params.ENV}
          echo "✅ Docker secret applied to ${params.ENV}"
        """
      }
    }

    stage('Deploy Database') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'DATABASE_ONLY'] } }
      steps {
        sh """
          set -e
          echo "Deploying Database for ${params.ENV}..."
          kubectl apply -f k8s/database-deployment.yaml -n ${params.ENV} || true
          for i in {1..24}; do
            READY=\$(kubectl get pod -l app=database -n ${params.ENV} -o jsonpath='{.items[0].status.containerStatuses[0].ready}' 2>/dev/null || echo "false")
            STATUS=\$(kubectl get pod -l app=database -n ${params.ENV} --no-headers -o custom-columns=STATUS:.status.phase 2>/dev/null || echo "Pending")
            echo "Database pod status: \$STATUS (ready: \$READY) (\$i/24)"
            [ "\$READY" = "true" ] && [ "\$STATUS" = "Running" ] && echo "Database is READY!" && break
            sleep 5
          done
          kubectl get svc -l app=database -n ${params.ENV}
        """
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
          docker build -t frontend:${IMAGE_TAG} ./frontend
          docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker rmi frontend:${IMAGE_TAG} || true
          docker image prune -f || true
          echo "✅ Frontend: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}"
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
          echo "✅ Backend: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}"
        """
      }
    }

    stage('Update & Commit Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
          sh """
            set -e
            echo "Updating Helm values..."
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${params.ENV}.yaml
            
            git config user.name "Thanuja"
            git config user.email "ratakondathanuja@gmail.com"
            git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml version.txt
            git commit -m "chore: images ${IMAGE_TAG} for ${params.ENV} [skip ci]" || echo "No changes"
            git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_10.git master
            echo "✅ Helm values updated!"
          """
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          sh """
            set -e
            echo "Applying ArgoCD for ${params.ENV}..."
            kubectl apply -f argocd/backend_${params.ENV}.yaml -n argocd
            kubectl apply -f argocd/frontend_${params.ENV}.yaml -n argocd
            kubectl apply -f argocd/database-app_${params.ENV}.yaml -n argocd
            
            sleep 5
            kubectl annotate application backend-${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            kubectl annotate application frontend-${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            kubectl annotate application database-${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            echo "✅ ArgoCD refreshed!"
          """
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        timeout(time: 3, unit: 'MINUTES') {
          sh """
            echo "=== 🎉 FINAL STATUS ${params.ENV} ==="
            kubectl get pods -n ${params.ENV} -o wide
            kubectl get svc -n ${params.ENV}
            kubectl get pvc -n ${params.ENV}
            kubectl get applications -n argocd | grep ${params.ENV}
            echo "✅ PIPELINE COMPLETE! Check ArgoCD UI"
          """
        }
      }
    }
  }

  post {
    always {
      sh """
        echo "=== FINAL CLEANUP ==="
        kubectl get ns ${params.ENV} || true
        kubectl get pvc -n ${params.ENV} || true
      """ || true
    }
    success {
      echo "🎉 SUCCESS! ${params.ENV} deployed with ${IMAGE_TAG}"
    }
    failure {
      echo "❌ FAILED - Debug PVC above"
      sh """
        kubectl describe pvc shared-pvc -n ${params.ENV} || true
        kubectl get events -n ${params.ENV} --sort-by=.lastTimestamp | tail -10 || true
      """ || true
    }
  }
}
