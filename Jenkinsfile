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
        echo "✅ Checked out code from ${GIT_REPO}"
      }
    }

    stage('Version Bump') {
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
            newVersion = "v1.${currentMinor + 1}.0"
            writeFile file: minorFile, text: "${currentMinor + 1}"
          } else if (params.VERSION_BUMP == 'major') {
            def majorFile = 'major_counter.txt'
            def currentMajor = fileExists(majorFile) ? readFile(majorFile).trim().toInteger() : 0
            newVersion = "v${currentMajor + 1}.0.0"
            writeFile file: majorFile, text: "${currentMajor + 1}"
          }

          env.IMAGE_TAG = newVersion
          writeFile file: 'version.txt', text: newVersion
          echo "🎯 ${params.VERSION_BUMP} → ${newVersion}"
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
            echo "\$HARBOR_PASS" | docker login ${REGISTRY} -u "\$HARBOR_USER" --password-stdin
            echo "✅ Docker login successful"
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          docker build -t ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} ./frontend
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          docker image prune -f
          echo "✅ Frontend: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}"
        """
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} ./backend
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          docker image prune -f
          echo "✅ Backend: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}"
        """
      }
    }

    stage('Update Helm Values & Commit') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          sh """
            echo "🔄 Updating Helm values for ${params.ENV}..."
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${params.ENV}.yaml
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${params.ENV}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${params.ENV}.yaml
            
            echo "📝 Helm values updated:"
            grep -E 'repository|tag' frontend-hc/frontendvalues_${params.ENV}.yaml
            grep -E 'repository|tag' backend-hc/backendvalues_${params.ENV}.yaml
          """
          
          withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git config user.name "Thanuja"
              git config user.email "ratakondathanuja@gmail.com"
              git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml version.txt
              git commit -m "chore: update images to ${IMAGE_TAG} for ${params.ENV} env" || echo "No changes to commit"
              git push https://\$GIT_USER:\$GIT_TOKEN@github.com/ThanujaRatakonda/kp_10.git master
              echo "✅ Pushed to GitHub → ArgoCD will auto-sync!"
            """
          }
        }
      }
    }

    stage('Apply ArgoCD Applications') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        sh """
          echo "🚀 Applying ArgoCD Applications for ${params.ENV}..."
          
          # Apply Docker registry secret (needed for image pulls)
          kubectl apply -f docker-registry-secret.yaml
          
          # Apply all ArgoCD apps (they auto-create namespaces)
          kubectl apply -f argocd/backend_${params.ENV}.yaml
          kubectl apply -f argocd/frontend_${params.ENV}.yaml
          kubectl apply -f argocd/database-app_${params.ENV}.yaml
          
          echo "🔄 Triggering hard refresh..."
          kubectl annotate application backend-${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          kubectl annotate application frontend-${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          kubectl annotate application database-${params.ENV} -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          
          echo "⏳ Waiting for ArgoCD sync (30s)..."
          sleep 30
          
          echo "📊 ArgoCD Application Status:"
          kubectl get applications -n argocd | grep ${params.ENV}
        """
      }
    }

    stage('Wait for Deployment') {
      steps {
        sh """
          echo "⏳ Waiting for pods to be ready in ${params.ENV}..."
          for i in {1..60}; do
            READY_PODS=\$(kubectl get pods -n ${params.ENV} -o jsonpath='{range .items[*]}{@.metadata.name}{":"}{@.status.containerStatuses[0].ready}{" "}{end}' 2>/dev/null | grep ":true" | wc -l || echo 0)
            TOTAL_PODS=\$(kubectl get pods -n ${params.ENV} --no-headers 2>/dev/null | wc -l || echo 0)
            echo "[\$i/60] \$READY_PODS/\$TOTAL_PODS pods ready"
            
            if [ "\$TOTAL_PODS" -gt 0 ] && [ "\$READY_PODS" = "\$TOTAL_PODS" ]; then
              echo "✅ All pods READY!"
              break
            fi
            sleep 10
          done
        """
      }
    }

    stage('Verify Deployment') {
      steps {
        sh """
          echo "=== 🎉 FINAL DEPLOYMENT STATUS ==="
          echo "📦 Namespace: ${params.ENV}"
          
          echo -e "\\n🐳 PODS:"
          kubectl get pods -n ${params.ENV} -o wide
          
          echo -e "\\n🌐 SERVICES:"
          kubectl get svc -n ${params.ENV}
          
          echo -e "\\n💾 STORAGE:"
          kubectl get pvc -n ${params.ENV}
          kubectl get pv | grep ${params.ENV}
          
          echo -e "\\n🔗 ARGOCD APPS:"
          kubectl get applications -n argocd | grep ${params.ENV} -A 3 -B 1
          
          echo -e "\\n📋 EVENTS (last 10):"
          kubectl get events -n ${params.ENV} --sort-by='.lastTimestamp' | tail -10
          
          echo -e "\\n✅ Images deployed: ${IMAGE_TAG}"
        """
      }
    }
  }
  
  post {
    always {
      sh """
        echo "=== PIPELINE SUMMARY ===
        ENV: ${params.ENV}
        ACTION: ${params.ACTION}
        IMAGE_TAG: ${IMAGE_TAG ?: 'N/A'}
        "
        kubectl get pods -n ${params.ENV} -o wide || true
      """
    }
    success {
      echo "🎉 PIPELINE SUCCESSFUL! Deployment completed for ${params.ENV}"
    }
    failure {
      echo "💥 PIPELINE FAILED!"
      sh """
        echo "🔍 Last 15 events in ${params.ENV}:"
        kubectl get events -n ${params.ENV} --sort-by='.lastTimestamp' | tail -15 || true
        echo "🔍 ArgoCD app status:"
        kubectl get applications -n argocd | grep ${params.ENV} || true
      """
    }
  }
}
