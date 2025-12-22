
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
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'git-creds', url: "${GIT_REPO}", branch: 'master'
      }
    }

    stage('Create Namespace if not exist') {
      steps {
        sh """
          set -e
          kubectl get namespace ${params.ENV} >/dev/null 2>&1 || kubectl create namespace ${params.ENV}
        """
      }
    }

    stage('Create StorageClass, PV, and PVC (static)') {
      steps {
        script {
          // Build environment-aware file names and PV names
          def PV_FILE = "k8s/shared-pv_${params.ENV}.yaml"
          def PVC_FILE = "k8s/shared-pvc_${params.ENV}.yaml"
          def PV_NAME = "shared-pv-${params.ENV}"

          sh """
            set -e

            # 1) StorageClass must exist first for static binding (no dynamic provisioning)
            kubectl get storageclass shared-storage >/dev/null 2>&1 || kubectl apply -f k8s/shared-storage-class.yaml

            # 2) PV is cluster-scoped (NO -n). Apply if not present.
            kubectl get pv ${PV_NAME} >/dev/null 2>&1 || kubectl apply -f ${PV_FILE}

            # 3) PVC is namespaced. The YAML already contains metadata.namespace (dev/qa),
            #    so DO NOT use -n with apply to avoid namespace mismatch conflicts.
            kubectl get pvc shared-pvc -n ${params.ENV} >/dev/null 2>&1 || kubectl apply -f ${PVC_FILE}

            # 4) Show binding status for visibility
            kubectl get pv ${PV_NAME}
            kubectl get pvc shared-pvc -n ${params.ENV}
            kubectl describe pvc shared-pvc -n ${params.ENV} | sed -n '1,80p'
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
        // Safer tag replacement (anchor line, avoid quoting the number)
        sh """
          set -e
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
          kubectl get application backend -n argocd >/dev/null 2>&1 || kubectl apply -f argocd/backend_${params.ENV}.yaml
          kubectl get application frontend -n argocd >/dev/null 2>&1 || kubectl apply -f argocd/frontend_${params.ENV}.yaml
          kubectl get application database -n argocd >/dev/null 2>&1 || kubectl apply -f argocd/database-app.yaml
        """
      }
    }
  }

  post {
    always {
      sh 'docker logout ${REGISTRY} || true'
    }
  }
}
