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
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'DATABASE_ONLY', 'ARGOCD_ONLY', 'MULTI_ENV'],
      description: 'Run full pipeline or specific components'
    )
    choice(
      name: 'ENVS',
      choices: ['dev', 'qa', 'dev-qa'], 
      description: 'Choose environment(s)'
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
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'MULTI_ENV'] } }
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

    stage('Create Namespaces') {
      steps {
        sh """
          kubectl get namespace ${params.ENVS == 'dev-qa' ? 'dev qa argocd' : params.ENVS} >/dev/null 2>&1 || \\
          kubectl create namespace ${params.ENVS == 'dev-qa' ? 'dev qa' : params.ENVS}
          kubectl get namespace argocd >/dev/null 2>&1 || kubectl create namespace argocd
          echo "Namespaces: ${params.ENVS} + argocd"
        """
      }
    }

    stage('Apply Docker Registry Secrets') {
      steps {
        script {
          def envs = params.ENVS == 'dev-qa' ? ['dev', 'qa'] : [params.ENVS]
          for (env in envs) {
            sh """
              kubectl apply -f dockersecret_${env}.yaml
              echo "regcred secret applied to ${env}"
            """
          }
        }
        sh """
          kubectl get secret regcred -n ${params.ENVS == 'dev-qa' ? 'dev qa' : params.ENVS}
        """
      }
    }

    stage('Apply Storage') {
      steps {
        timeout(time: 8, unit: 'MINUTES') {
          script {
            def envs = params.ENVS == 'dev-qa' ? ['dev', 'qa'] : [params.ENVS]
            for (envName in envs) {
              sh """
                ENV_NS="${envName}"
                PV_NAME="shared-pv-\$ENV_NS"
                PVC_NAME="shared-pvc"
                
                PVC_STATUS=\$(kubectl get pvc \$PVC_NAME -n \$ENV_NS -o jsonpath='{.status.phase}' 2>/dev/null || echo "None")
                if [ "\$PVC_STATUS" = "Bound" ]; then
                  echo " PVC \$PVC_NAME already BOUND in \$ENV_NS → SKIPPING!"
                  kubectl get pv \$PV_NAME
                  kubectl get pvc \$PVC_NAME -n \$ENV_NS
                  exit 0
                fi
                
                echo "PVC not bound, fixing storage for ${envName}..."
                kubectl patch pv \$PV_NAME -p '{"spec":{"claimRef":null}}' || true
                kubectl delete pvc \$PVC_NAME -n \$ENV_NS --force --grace-period=0 || true
                sleep 5
                
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
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'MULTI_ENV'] } }
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
            echo "Docker login successful"
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'MULTI_ENV'] } }
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
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY', 'MULTI_ENV'] } }
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
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'MULTI_ENV'] } }
      steps {
        script {
          def envs = params.ENVS == 'dev-qa' ? ['dev', 'qa'] : [params.ENVS]
          for (envName in envs) {
            sh """
              set -e
              echo "Updating Helm values for ${envName}..."
              sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${envName}.yaml
              sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${envName}.yaml
              sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${envName}.yaml
              sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${envName}.yaml
            """
          }

          withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git config user.name "Thanuja"
              git config user.email "ratakondathanuja@gmail.com"
              git add frontend-hc/frontendvalues_*.yaml backend-hc/backendvalues_*.yaml version.txt
              git commit -m "chore: images ${IMAGE_TAG} for ${params.ENVS}" || echo "No changes"
              git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_11.git master
            """
          }
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY', 'MULTI_ENV'] } }
      steps {
        script {
          def envs = params.ENVS == 'dev-qa' ? ['dev', 'qa'] : [params.ENVS]
          for (envName in envs) {
              kubectl apply -f argocd/backend_${envName}.yaml
              kubectl apply -f argocd/frontend_${envName}.yaml
              kubectl apply -f argocd/database-app_${envName}.yaml
              
              kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              echo " ${envName} ArgoCD refreshed"
            """
            sleep 10
          }
        }
        sh """
          echo " ArgoCD refreshed for ${params.ENVS}"
        """
      }
    }

    stage('Verify All Deployments') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          sh """

            kubectl get pods -n ${params.ENVS == 'dev-qa' ? 'dev qa' : params.ENVS} -o wide
            
            kubectl get svc -n ${params.ENVS == 'dev-qa' ? 'dev qa' : params.ENVS}
            
            kubectl get applications -n argocd -o custom-columns="NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status" | grep -E "(backend|frontend|database)"
            
            kubectl get secret regcred -n ${params.ENVS == 'dev-qa' ? 'dev qa' : params.ENVS}
            
            kubectl get events -n ${params.ENVS == 'dev-qa' ? 'dev qa' : params.ENVS} --sort-by='.lastTimestamp' | tail -10
          """
        }
      }
    }
  }
}
