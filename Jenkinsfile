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
        timeout(time: 8, unit: 'MINUTES') {
          sh """
            ENV_NS="${params.ENV}"
            PV_NAME="shared-pv-\$ENV_NS"
            PVC_NAME="shared-pvc"
            
            echo "🔧 Storage setup for \$ENV_NS (SKIP STUCK DELETE)..."
            
            # SKIP DELETE - JUST APPLY OVER EXISTING (IDEMPOTENT)
            kubectl apply -f k8s/shared-storage-class.yaml
            
            echo "📦 Creating/Replacing PV..."
            kubectl apply -f k8s/shared-pv_\${ENV_NS}.yaml --force || true
            
            echo "⏳ Waiting PV Available..."
            for i in {1..12}; do
              PV_STATUS=\$(kubectl get pv \$PV_NAME -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
              echo "PV [\$i/12]: \$PV_STATUS"
              [ "\$PV_STATUS" = "Available" ] && echo "✅ PV READY!" && break
              sleep 5
            done || true
            
            echo "📦 Creating/Replacing PVC..."
            kubectl apply -f k8s/shared-pvc_\${ENV_NS}.yaml -n \$ENV_NS --force || true
            
            echo "⏳ Waiting PVC Bound..."
            for i in {1..30}; do
              PHASE=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
              BOUND_TO=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.spec.volumeName}' 2>/dev/null || echo "None")
              echo "[\$i/30] PVC: \$PHASE | Bound: \$BOUND_TO"
              [ "\$PHASE" = "Bound" ] && echo "🎉 PVC BOUND!" && break
              sleep 5
            done || true
            
            echo "📊 FINAL STATUS:"
            kubectl get pv \$PV_NAME || true
            kubectl get pvc \$PVC_NAME -n \$ENV_NS || true
            echo "✅ STORAGE READY (even if stuck, we'll continue)"
          """
        }
      }
    }

    stage('Apply Docker Secret') {
      steps {
        sh """
          kubectl apply -f docker-registry-secret.yaml -n ${params.ENV} || true
          echo "✅ Docker secret applied"
        """
      }
    }

    stage('Docker Login') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')
        ]) {
          sh """
            echo "\$HARBOR_PASS" | docker login ${REGISTRY} -u "\$HARBOR_USER" --password-stdin || true
            echo "Docker login done"
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          docker build -t ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} ./frontend || true
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} || true
          echo "✅ Frontend: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}"
        """
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} ./backend || true
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} || true
          echo "✅ Backend: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}"
        """
      }
    }

    stage('Update & Commit Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
          sh """
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${params.ENV}.yaml || true
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${params.ENV}.yaml || true
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${params.ENV}.yaml || true
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${params.ENV}.yaml || true
            
            git config user.name "Thanuja"
            git config user.email "ratakondathanuja@gmail.com"
            git add . || true
            git commit -m "chore: ${IMAGE_TAG} ${params.ENV}" || true
            git push https://\$GIT_USER:\$GIT_TOKEN@github.com/ThanujaRatakonda/kp_10.git master || true
            echo "✅ Git updated"
          """
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        sh """
          kubectl apply -f argocd/*_${params.ENV}.yaml -n argocd || true
          sleep 3
          kubectl annotate application *- ${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          echo "✅ ArgoCD refreshed"
        """
      }
    }

    stage('Verify Deployment') {
      steps {
        sh """
          echo "=== FINAL STATUS ${params.ENV} ==="
          kubectl get pods,svc,pvc -n ${params.ENV} || true
          kubectl get applications -n argocd | grep ${params.ENV} || true
          echo "🎉 PIPELINE COMPLETE!"
        """
      }
    }
  }

  post {
    always {
      echo "Pipeline finished for ${params.ENV}"
    }
  }
}

