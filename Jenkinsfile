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
        sh 'kubectl config current-context && kubectl version --short'
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
            kubectl get namespace ${params.ENV} -o yaml
            echo "✅ Namespace ${params.ENV} ready!"
          """
        }
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
            
            echo "🔧 Setting up storage for \$ENV_NS..."
            echo "📍 Current context: \$(kubectl config current-context)"
            
            # Clean existing resources
            echo "🧹 Cleaning existing PV/PVC..."
            kubectl delete pvc \$PVC_NAME -n \$ENV_NS --ignore-not-found=true --force --grace-period=0 || true
            kubectl delete pv \$PV_NAME --ignore-not-found=true --force --grace-period=0 || true
            sleep 5
            
            # Apply storage resources in correct order
            echo "📦 Applying StorageClass..."
            kubectl apply -f k8s/shared-storage-class.yaml || { echo "❌ StorageClass failed"; exit 1; }
            
            echo "📦 Applying PV..."
            kubectl apply -f k8s/shared-pv_\${ENV_NS}.yaml || { echo "❌ PV failed"; exit 1; }
            kubectl wait --for=condition=Available pv/\$PV_NAME --timeout=2m || { echo "❌ PV not available"; exit 1; }
            
            echo "📦 Applying PVC..."
            kubectl apply -f k8s/shared-pvc_\${ENV_NS}.yaml -n \$ENV_NS || { echo "❌ PVC failed"; exit 1; }
            
            # Wait for PVC to bind with kubectl wait (more reliable)
            echo "⏳ Waiting for PVC to bind (max 5m)..."
            kubectl wait --for=condition=Bound pvc/\$PVC_NAME -n \$ENV_NS --timeout=5m || { 
              echo "❌ PVC failed to bind. Debug info:" 
              kubectl describe pvc \$PVC_NAME -n \$ENV_NS 
              kubectl describe pv \$PV_NAME 
              kubectl get events -n \$ENV_NS --sort-by=.lastTimestamp | tail -20
              exit 1 
            }
            
            echo "✅ Storage ready!"
            kubectl get pv \$PV_NAME -o wide
            kubectl get pvc \$PVC_NAME -n \$ENV_NS -o wide
          """
        }
      }
    }

    stage('Verify Docker Secret') {
      steps {
        timeout(time: 30, unit: 'SECONDS') {
          sh """
            kubectl get secret regcred -n ${params.ENV} >/dev/null 2>&1 && echo '✅ Docker secret exists' || {
              echo '⚠️ Docker secret missing, will create in next step'
            }
          """
        }
      }
    }

    stage('Docker Login') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
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
          echo "✅ Frontend pushed: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}"
        """
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} ./backend
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          echo "✅ Backend pushed: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}"
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
            git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml version.txt
            git commit -m "chore: update images to ${IMAGE_TAG} for ${params.ENV} [skip ci]" || echo "No changes to commit"
            git push https://\$GIT_USER:\$GIT_TOKEN@github.com/ThanujaRatakonda/kp_10.git HEAD:master
            echo "✅ Helm values updated and pushed"
          """
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          sh """
            echo "🔄 Applying ArgoCD applications for ${params.ENV}..."
            kubectl apply -f argocd/backend_${params.ENV}.yaml
            kubectl apply -f argocd/frontend_${params.ENV}.yaml
            kubectl apply -f argocd/database-app_${params.ENV}.yaml
            
            echo "🔄 Hard refresh ArgoCD applications..."
            kubectl patch app frontend -n argocd -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' --type=merge || true
            kubectl patch app backend -n argocd -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' --type=merge || true
            kubectl patch app database -n argocd -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' --type=merge || true
            
            echo "✅ ArgoCD applications refreshed!"
          """
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        timeout(time: 3, unit: 'MINUTES') {
          sh """
            echo "=== 🚀 FINAL STATUS ${params.ENV} ==="
            echo "📦 Pods:"
            kubectl get pods -n ${params.ENV} -o wide
            
            echo "🌐 Services:"
            kubectl get svc -n ${params.ENV}
            
            echo "💾 Storage:"
            kubectl get pvc -n ${params.ENV}
            
            echo "🔗 ArgoCD Apps:"
            kubectl get applications -n argocd | grep -E "(frontend|backend|database)" || echo "No matching apps found"
            
            echo "🎉 PIPELINE COMPLETE! Check ArgoCD UI for deployment status"
          """
        }
      }
    }
  }

  post {
    always {
      script {
        try {
          sh "kubectl get ns ${params.ENV} -o yaml || true"
          sh "kubectl get pvc -n ${params.ENV} || true"
        } catch (Exception e) {
          echo "Post cleanup: ${e.getMessage()}"
        }
      }
    }
    success {
      echo "🎉 SUCCESS! Deployed ${params.ENV} with ${IMAGE_TAG}"
    }
    failure {
      echo "❌ PIPELINE FAILED - Check storage/PV logs above"
      script {
        try {
          sh """
            echo "=== FAILURE DEBUG ==="
            kubectl get pvc -n ${params.ENV} -o yaml || true
            kubectl get events -n ${params.ENV} --sort-by=.lastTimestamp | tail -10 || true
          """
        } catch (Exception e) {}
      }
    }
  }
}

