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
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'git-creds', url: "${GIT_REPO}", branch: 'master'
      }
    }

    stage('Read & Auto-Increment Version') {
      steps {
        script {
          def versionFile = 'version.txt'
          def newVersion
          
          if (fileExists(versionFile)) {
            def currentVersion = readFile(versionFile).trim()
            echo "Current version: ${currentVersion}"
            
            // Parse semantic version (v1.0.0 -> 1.0.0)
            def parts = currentVersion.replaceAll(/^v/, '').tokenize('.')
            if (parts.size() == 3) {
              def major = parts[0].toInteger()
              def minor = parts[1].toInteger()
              def patch = parts[2].toInteger()
              
              if (patch >= 9) {
                // v1.0.9+ → v1.1.0 (reset patch to 0, increment minor)
                newVersion = "v${major}.${minor + 1}.0"
              } else {
                // Normal increment: v1.0.5 → v1.0.6
                newVersion = "v${major}.${minor}.${patch + 1}"
              }
            } else {
              newVersion = "v1.0.1"
            }
          } else {
            newVersion = "v1.0.0"
          }
          
          env.IMAGE_TAG = newVersion
          writeFile file: versionFile, text: newVersion
          echo " New version: ${env.IMAGE_TAG}"
        }
      }
    }

    stage('Create Namespace') {
      steps {
        script {
          sh """
            kubectl get namespace ${params.ENV} >/dev/null 2>&1 || kubectl create namespace ${params.ENV}
            echo "Namespace ${params.ENV} ready"
          """
        }
      }
    }

    stage('Apply Storage') {
      steps {
        script {
          def ENV_NS = params.ENV
          def PV_FILE = "k8s/shared-pv_${ENV_NS}.yaml"
          def PVC_FILE = "k8s/shared-pvc_${ENV_NS}.yaml"
          
          sh """
            set -e
            echo "Applying storage for ${ENV_NS}..."
            kubectl apply -f k8s/shared-storage-class.yaml || true
            test -f ${PV_FILE} && kubectl apply -f ${PV_FILE}
            kubectl get pv shared-pv-${ENV_NS}
            test -f ${PVC_FILE} && kubectl apply -f ${PVC_FILE}
            
            for i in {1..30}; do
              PHASE=\$(kubectl get pvc shared-pvc -n ${ENV_NS} -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
              [ "\$PHASE" = "Bound" ] && echo "PVC Bound!" && break
              echo "PVC: \$PHASE (\$i/30)"
              sleep 5
            done
          """
        }
      }
    }
stage('Apply Docker Secret') {
  steps {
    sh """
      kubectl apply -f docker-registry-secret.yaml
      echo "Docker secret applied for ${params.ENV}"
    """
  }
}
    stage('Deploy Database') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'DATABASE_ONLY'] } }
      steps {
        script {
          sh """
            set -e
            echo "Deploying Database to ${params.ENV}..."
            helm upgrade --install database-database-hc ./database-hc \\
              --namespace ${params.ENV} --values database-hc/databasevalues_${params.ENV}.yaml
            kubectl rollout status sts/database-database-hc -n ${params.ENV} --timeout=300s
            echo "Database ready!"
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        script {
          sh """
            set -e
            docker build -t frontend:${IMAGE_TAG} ./frontend
            echo "\${DOCKER_PASSWORD}" | docker login ${REGISTRY} -u ${DOCKER_USERNAME} --password-stdin
            docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
            echo "Frontend: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}"
          """
        }
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        script {
          sh """
            set -e
            docker build -t backend:${IMAGE_TAG} ./backend
            echo "\${DOCKER_PASSWORD}" | docker login ${REGISTRY} -u ${DOCKER_USERNAME} --password-stdin
            docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
            echo "Backend: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}"
          """
        }
      }
    }

    stage('Update & Commit Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          sh """
            set -e
            echo "Updating Helm values..."
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
              git commit -m "chore: images ${IMAGE_TAG} for ${params.ENV}" || echo "No changes"
              git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_10.git master
            """
          }
        }
      }
    }

    stage('Apply ArgoCD Apps') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        script {
          sh """
            set -e
            echo "Applying ArgoCD for ${params.ENV}..."
            kubectl apply -f argocd/backend_${params.ENV}.yaml
            kubectl apply -f argocd/frontend_${params.ENV}.yaml
            kubectl apply -f argocd/database-app_${params.ENV}.yaml
            
            kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          """
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        script {
          sh """
            echo "=== FINAL STATUS ==="
            kubectl get pods -n ${params.ENV}
            kubectl get svc -n ${params.ENV}
            kubectl get applications -n argocd
          """
        }
      }
    }
  }
}
