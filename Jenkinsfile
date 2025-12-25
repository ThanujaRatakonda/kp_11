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
          kubectl get namespace argocd >/dev/null 2>&1 || kubectl create namespace argocd
          echo "✅ Namespaces: ${params.ENV} + argocd"
        """
      }
    }

    stage('Apply Storage') {
  steps {
    timeout(time: 8, unit: 'MINUTES') {
      sh """
        ENV_NS="${params.ENV}"
        echo "🔧 Storage for \$ENV_NS..."
        
        # 🔥 QA FIX: Clean stuck Released PV (dev skips this naturally)
        if [ "\$ENV_NS" = "qa" ]; then
          echo "🔥 QA: Clearing stuck PV..."
          kubectl patch pv shared-pv-\$ENV_NS -p '{"spec":{"claimRef":null}}' || true
          kubectl delete pvc shared-pvc -n \$ENV_NS --ignore-not-found=true || true
          sleep 5
        fi
        
        # Apply fresh
        kubectl apply -f k8s/shared-storage-class.yaml || true
        kubectl apply -f k8s/shared-pv_\${ENV_NS}.yaml || true
        sleep 3
        kubectl apply -f k8s/shared-pvc_\${ENV_NS}.yaml -n \$ENV_NS || true
        
        # Wait PV Available
        echo "⏳ PV Available..."
        for i in {1..12}; do
          PV_STATUS=\$(kubectl get pv shared-pv-\$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
          echo "PV[\$i/12]: \$PV_STATUS"
          [ "\$PV_STATUS" = "Available" ] && break || sleep 5
        done
        
        # Wait PVC Bound
        echo "⏳ PVC Bound..."
        for i in {1..30}; do
          PHASE=\$(kubectl get pvc shared-pvc -n \$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
          echo "PVC[\$i/30]: \$PHASE"
          [ "\$PHASE" = "Bound" ] && echo "✅ PVC BOUND!" && break || sleep 5
        done
        
        kubectl get pv shared-pv-\$ENV_NS
        kubectl get pvc shared-pvc -n \$ENV_NS
        echo "✅ STORAGE PERFECT!"
      """
    }
  }
}

    // ... ALL OTHER STAGES EXACTLY AS YOU HAVE ...
    stage('Apply Docker Secret') {
      steps {
        sh "kubectl apply -f docker-registry-secret.yaml || true && echo '✅ Docker secret OK'"
      }
    }

    stage('Docker Login') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
          sh "echo \"\$HARBOR_PASS\" | docker login ${REGISTRY} -u \"\$HARBOR_USER\" --password-stdin"
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          docker build -t ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} ./frontend
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          echo "✅ Frontend pushed"
        """
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} ./backend
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          echo "✅ Backend pushed"
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
            git add .
            git commit -m "chore: ${IMAGE_TAG} ${params.ENV}" || true
            git push https://\$GIT_USER:\$GIT_TOKEN@github.com/ThanujaRatakonda/kp_10.git master
          """
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        sh """
          kubectl apply -f argocd/backend_${params.ENV}.yaml
          kubectl apply -f argocd/frontend_${params.ENV}.yaml
          kubectl apply -f argocd/database-app_${params.ENV}.yaml
          
          kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          echo "✅ ArgoCD refreshed"
        """
      }
    }

    stage('Verify Deployment') {
      steps {
        sh """
          echo "=== FINAL STATUS ${params.ENV} ==="
          kubectl get pods -n ${params.ENV}
          kubectl get svc -n ${params.ENV}
          kubectl get applications -n argocd | grep -E "(backend|frontend|database)"
        """
      }
    }
  }
}
