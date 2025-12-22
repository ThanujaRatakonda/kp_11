pipeline {
  agent any

  environment {
    REGISTRY = "10.131.103.92:8090"
    PROJECT  = "kp_10"
    IMAGE_TAG = "${BUILD_NUMBER}"
    GIT_REPO = "https://github.com/ThanujaRatakonda/kp_10.git"
    DOCKER_USERNAME = "admin"
    DOCKER_PASSWORD = "Harbor12345"
  }

  parameters {
    choice(
      name: 'ACTION',
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'DATABASE_ONLY', 'ARGOCD_ONLY'],
      description: 'Run full pipeline, only frontend/backend/database, or just apply ArgoCD resources'
    )
    choice(
      name: 'ENV',
      choices: ['dev', 'qa'],
      description: 'Choose the environment to deploy (dev/qa)'
    )
    booleanParam(
      name: 'RESET_STORAGE',
      defaultValue: false,
      description: 'If true, delete PV/PVC for this ENV before re-applying'
    )
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'git-creds', url: "${GIT_REPO}", branch: 'master'
      }
    }

    stage('Create Namespace') {
      steps {
        script {
          def ENV_NS = params.ENV
          sh """
            kubectl get namespace ${ENV_NS} >/dev/null 2>&1 || kubectl create namespace ${ENV_NS}
            echo "Namespace ${ENV_NS} is ready."
          """
        }
      }
    }

    stage('Reset Storage (optional)') {
      when { expression { params.RESET_STORAGE } }
      steps {
        script {
          def ENV_NS = params.ENV
          def PV_NAME = "shared-pv-${ENV_NS}"
          sh """
            set -e
            echo "RESET_STORAGE=true: cleaning PV/PVC for ENV=${ENV_NS}"
            kubectl scale deploy backend-backend-hc -n ${ENV_NS} --replicas=0 || true
            kubectl scale deploy frontend-frontend-hc -n ${ENV_NS} --replicas=0 || true
            kubectl scale sts database-database-hc -n ${ENV_NS} --replicas=0 || true
            kubectl patch pvc shared-pvc -n ${ENV_NS} -p '{\"metadata\":{\"finalizers\":[]}}' || true
            kubectl delete pvc shared-pvc -n ${ENV_NS} --force --grace-period=0 || true
            kubectl patch pv ${PV_NAME} -p '{\"metadata\":{\"finalizers\":[]}}' || true
            kubectl delete pv ${PV_NAME} --force --grace-period=0 || true
          """
        }
      }
    }

    stage('Apply StorageClass, PV, PVC') {
      steps {
        script {
          def ENV_NS = params.ENV
          def PV_FILE = "k8s/shared-pv_${ENV_NS}.yaml"
          def PVC_FILE = "k8s/shared-pvc_${ENV_NS}.yaml"
          sh """
            set -e
            kubectl apply -f k8s/shared-storage-class.yaml || true
            test -f ${PV_FILE} && kubectl apply -f ${PV_FILE}
            test -f ${PVC_FILE} && kubectl apply -f ${PVC_FILE}
            for i in {1..30}; do
              PHASE=\$(kubectl get pvc shared-pvc -n ${ENV_NS} -o jsonpath='{.status.phase}' 2>/dev/null || echo "Pending")
              [ "\$PHASE" = "Bound" ] && echo "PVC Bound ✅" && break
              echo "PVC: \$PHASE (attempt \$i)"
              sleep 5
            done
          """
        }
      }
    }

    stage('Create Docker Registry Secret') {
      steps {
        script {
          def ENV_NS = params.ENV
          sh """
            kubectl get secret regcred -n ${ENV_NS} >/dev/null 2>&1 || \\
            kubectl create secret docker-registry regcred -n ${ENV_NS} \\
              --docker-server=${REGISTRY} --docker-username=${DOCKER_USERNAME} --docker-password=${DOCKER_PASSWORD}
          """
        }
      }
    }

    // ✅ FIXED: Deploy Database using YOUR Helm chart + values
    stage('Deploy Database') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'DATABASE_ONLY'] } }
      steps {
        script {
          def ENV_NS = params.ENV
          sh """
            set -e
            echo "🚀 Deploying Database to ${ENV_NS} using Helm..."
            helm upgrade --install database-database-hc database-hc \\
              --namespace ${ENV_NS} \\
              --values database-hc/databasevalues_${ENV_NS}.yaml
            
            echo "⏳ Waiting for database..."
            kubectl rollout status sts/database-database-hc -n ${ENV_NS} --timeout=300s
            
            echo "✅ Database ready!"
            kubectl get sts,pvc,svc -n ${ENV_NS} | grep database
          """
        }
      }
    }

    stage('Build & Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        script {
          sh """
            docker build -t frontend:${IMAGE_TAG} ./frontend
            echo "\${DOCKER_PASSWORD}" | docker login ${REGISTRY} -u ${DOCKER_USERNAME} --password-stdin
            docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Build & Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        script {
          sh """
            docker build -t backend:${IMAGE_TAG} ./backend
            echo "\${DOCKER_PASSWORD}" | docker login ${REGISTRY} -u ${DOCKER_USERNAME} --password-stdin
            docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Update Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        script {
          def ENV_NS = params.ENV
          sh """
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/frontend|' frontend-hc/frontendvalues_${ENV_NS}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' frontend-hc/frontendvalues_${ENV_NS}.yaml
            sed -i 's|repository:.*|repository: ${REGISTRY}/${PROJECT}/backend|' backend-hc/backendvalues_${ENV_NS}.yaml
            sed -i 's|tag:.*|tag: ${IMAGE_TAG}|' backend-hc/backendvalues_${ENV_NS}.yaml
          """
        }
      }
    }

    stage('Commit Helm Changes') {
      steps {
        script {
          def ENV_NS = params.ENV
          withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git config user.name "Thanuja"
              git config user.email "ratakondathanuja@gmail.com"
              git add frontend-hc/frontendvalues_${ENV_NS}.yaml backend-hc/backendvalues_${ENV_NS}.yaml
              git commit -m "chore: update images ${IMAGE_TAG} for ${ENV_NS}" || true
              git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_10.git master
            """
          }
        }
      }
    }

    // ✅ FIXED: Apply YOUR ArgoCD files (database-app_${ENV_NS}.yaml)
    stage('Apply ArgoCD Resources') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY', 'DATABASE_ONLY'] } }
      steps {
        script {
          def ENV_NS = params.ENV
          sh """
            set -e
            kubectl apply -f argocd/backend_${ENV_NS}.yaml
            kubectl apply -f argocd/frontend_${ENV_NS}.yaml
            kubectl apply -f argocd/database-app_${ENV_NS}.yaml  # ✅ YOUR FILE!
            
            # Force refresh
            kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            
            kubectl get applications -n argocd
          """
        }
      }
    }
  }

  post {
    always {
      sh 'docker logout ${REGISTRY} || true'
      sh 'docker image prune -f || true'
    }
  }
}
