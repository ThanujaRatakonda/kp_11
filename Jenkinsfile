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
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'ARGOCD_ONLY'],
      description: 'Run full pipeline, only frontend/backend, or just apply ArgoCD resources'
    )
    choice(
      name: 'ENV',
      choices: ['dev', 'qa'],
      description: 'Choose the environment to deploy (dev/qa)'
    )
    booleanParam(
      name: 'RESET_STORAGE',
      defaultValue: false,
      description: 'If true, delete PV/PVC for this ENV before re-applying (unsticks Terminating/finalizers)'
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
        sh """
          set -e
          kubectl get namespace ${params.ENV} >/dev/null 2>&1 || kubectl create namespace ${params.ENV}
          echo "Namespace ${params.ENV} is ready."
        """
      }
    }

    stage('Reset Storage (optional)') {
      when { expression { params.RESET_STORAGE } }
      steps {
        script {
          def PV_NAME = "shared-pv-${params.ENV}"
          sh """
            set -e
            echo "RESET_STORAGE=true: cleaning PV/PVC for ENV=${params.ENV}"

            # Scale down apps that might hold the PVC (ignore errors)
            kubectl scale deploy backend-backend-hc -n ${params.ENV} --replicas=0 || true
            kubectl scale sts database-database-hc -n ${params.ENV} --replicas=0 || true
            kubectl get pods -n ${params.ENV} || true

            # Remove PVC finalizers if stuck
            kubectl patch pvc shared-pvc -n ${params.ENV} -p '{"metadata":{"finalizers":[]}}' || true
            kubectl delete pvc shared-pvc -n ${params.ENV} --force --grace-period=0 || true

            # Remove PV finalizers if stuck
            kubectl patch pv ${PV_NAME} -p '{"metadata":{"finalizers":[]}}' || true
            kubectl delete pv ${PV_NAME} --force --grace-period=0 || true
          """
        }
      }
    }

    stage('Apply StorageClass, PV, PVC (static)') {
      steps {
        script {
          def PV_FILE = "k8s/shared-pv_${params.ENV}.yaml"     // e.g., shared-pv_dev.yaml / shared-pv_qa.yaml
          def PVC_FILE = "k8s/shared-pvc_${params.ENV}.yaml"   // e.g., shared-pvc_dev.yaml / shared-pvc_qa.yaml
          def PV_NAME = "shared-pv-${params.ENV}"

          sh """
            set -e

            # 1) StorageClass (static) — must exist before PV/PVC
            echo "Applying StorageClass: shared-storage"
            kubectl apply -f k8s/shared-storage-class.yaml
            kubectl get storageclass shared-storage

            # 2) PV is cluster-scoped — always apply (idempotent)
            echo "Applying PV: ${PV_NAME}"
            kubectl apply -f ${PV_FILE}
            kubectl get pv ${PV_NAME}

            # 3) PVC — YAML contains metadata.namespace, do NOT use -n for apply
            echo "Applying PVC: shared-pvc (namespace=${params.ENV})"
            kubectl apply -f ${PVC_FILE}
            kubectl get pvc shared-pvc -n ${params.ENV}

            # 4) Wait until PVC is Bound
            echo "Waiting for PVC shared-pvc to become Bound..."
            for i in {1..24}; do
              PHASE=$(kubectl get pvc shared-pvc -n ${params.ENV} -o jsonpath='{.status.phase}' || echo "")
              if [ "$PHASE" = "Bound" ]; then
                echo "PVC is Bound ✅"
                break
              fi
              echo "PVC phase: $PHASE (attempt $i/24)"
              sleep 5
            done

            echo "PVC status:"
            kubectl describe pvc shared-pvc -n ${params.ENV} | sed -n '1,120p'
            echo "PV status:"
            kubectl describe pv ${PV_NAME} | sed -n '1,80p'
          """
        }
      }
    }

    stage('Create Docker Registry Secret') {
      steps {
        sh """
          set -e
          kubectl get secret regcred -n ${params.ENV} >/dev/null 2>&1 || kubectl create secret docker-registry regcred -n ${params.ENV} \
            --docker-server=${REGISTRY} \
            --docker-username=${DOCKER_USERNAME} \
            --docker-password=${DOCKER_PASSWORD}
          kubectl get secret regcred -n ${params.ENV}
        """
      }
    }

    /* =========================
       FRONTEND
       ========================= */
    stage('Build Frontend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh "docker build -t frontend:${IMAGE_TAG} ./frontend"
      }
    }

    stage('Push Frontend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'harbor-creds',
          usernameVariable: 'USER',
          passwordVariable: 'PASS'
        )]) {
          sh """
            set -e
            echo "$PASS" | docker login ${REGISTRY} -u "$USER" --password-stdin
            docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Update Frontend Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          set -e
          # Update repository and tag in values file
          sed -i -E 's#^([[:space:]]*repository:[[:space:]]*).*$#\\1${REGISTRY}/${PROJECT}/frontend#' frontend-hc/frontendvalues_${params.ENV}.yaml
          sed -i -E 's/^([[:space:]]*tag:[[:space:]]*).*/\\1${IMAGE_TAG}/' frontend-hc/frontendvalues_${params.ENV}.yaml
        """
      }
    }

    /* =========================
       BACKEND
       ========================= */
    stage('Build Backend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh "docker build -t backend:${IMAGE_TAG} ./backend"
      }
    }

    stage('Push Backend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'harbor-creds',
          usernameVariable: 'USER',
          passwordVariable: 'PASS'
        )]) {
          sh """
            set -e
            echo "$PASS" | docker login ${REGISTRY} -u "$USER" --password-stdin
            docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Update Backend Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          set -e
          # Update repository and tag in values file
          sed -i -E 's#^([[:space:]]*repository:[[:space:]]*).*$#\\1${REGISTRY}/${PROJECT}/backend#' backend-hc/backendvalues_${params.ENV}.yaml
          sed -i -E 's/^([[:space:]]*tag:[[:space:]]*).*/\\1${IMAGE_TAG}/' backend-hc/backendvalues_${params.ENV}.yaml
        """
      }
    }

    /* =========================
       COMMIT FOR ARGO CD
       ========================= */
    stage('Commit & Push Helm Changes') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'GitHub',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          sh """
            set -e
            git config user.name "Thanuja"
            git config user.email "ratakondathanuja@gmail.com"
            git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml
            git commit -m "Update images to tag ${IMAGE_TAG}" || echo "No changes"
            git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_10.git master
          """
        }
      }
    }

    /* =========================
       Apply ArgoCD Application manifests
       ========================= */
    stage('Apply ArgoCD Resources') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY'] } }
      steps {
        sh """
          set -e
          kubectl apply -f argocd/backend_${params.ENV}.yaml
          kubectl apply -f argocd/frontend_${params.ENV}.yaml
          kubectl apply -f argocd/database-app.yaml

          # Force refresh to avoid Unknown status due to cache
          kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          kubectl annotate application backend  -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
          kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true

          kubectl get application -n argocd
        """
      }
    }
  }

  post {
    always {
      sh '''
        docker logout ${REGISTRY} || true
        docker image prune -f || true
      '''
    }
  }
}
