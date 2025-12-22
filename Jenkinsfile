pipeline {
  agent any

  environment {
    REGISTRY = "10.131.103.92:8090"
    PROJECT  = "kp_9"
    IMAGE_TAG = "${BUILD_NUMBER}"
    GIT_REPO = "https://github.com/ThanujaRatakonda/kp_9.git"
  }

  parameters {
    choice(
      name: 'ACTION',
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'ARGOCD_ONLY'],
      description: 'Run full pipeline, only frontend/backend, or just apply ArgoCD resources'
    )
    // Removed ENV choice as dev is already hardcoded
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'git-creds', url: "${GIT_REPO}", branch: 'master'
      }
    }

    /* =========================
       FRONTEND
       ========================= */
    stage('Build Frontend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          docker build -t frontend:${IMAGE_TAG} ./frontend
        """
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
            docker login ${REGISTRY} -u $USER -p $PASS
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
          sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' frontend-hc/frontendvalues.yaml
        """
      }
    }

    /* =========================
       BACKEND
       ========================= */
    stage('Build Backend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t backend:${IMAGE_TAG} ./backend
        """
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
            docker login ${REGISTRY} -u $USER -p $PASS
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
          sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' backend-hc/backendvalues.yaml
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
            git config user.name "thanuja"
            git config user.email "ratakondathanuja@gmail.com"
            git add frontend-hc/frontendvalues.yaml backend-hc/backendvalues.yaml
            git commit -m "Update images to tag ${IMAGE_TAG}" || echo "No changes"
            git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_9.git master
          """
        }
      }
    }

    /* =========================
       CREATE PVC & BIND PV
       ========================= */
    stage('Create PVC & Bind PV') {
      steps {
        script {
          // Ensure namespace "dev" exists
          sh "kubectl get namespace dev || kubectl create namespace dev"

          // Apply the PVC for shared storage
          sh "kubectl apply -f k8s/shared-pvc.yaml -n dev"

          // Wait for the PVC to be bound to the PV
          sh """
            while [[ $(kubectl get pvc shared-pvc -n dev -o=jsonpath='{.status.phase}') != "Bound" ]]; do
              echo "Waiting for PVC to bind to PV..."
              sleep 5
            done
          """
        }
      }
    }

    /* =========================
       APPLY K8S AND ARGOCD RESOURCES
       ========================= */
    stage('Apply Kubernetes & ArgoCD Resources') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY'] } }
      steps {
        script {
          // Apply Kubernetes resources (PV, PVC, etc.)
          sh """
            kubectl apply -f k8s/ -n dev
          """

          // Apply ArgoCD resources
          sh """
            kubectl apply -f argocd/
          """
        }
      }
    }
  }
}
