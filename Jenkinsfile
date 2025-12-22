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
      choices: ['qa', 'pro'],  // Only qa and pro environments
      description: 'Choose the environment to deploy (qa/pro)'
    )
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'git-creds', url: "${GIT_REPO}", branch: 'master'
      }
    }

    /* =========================
       Create Docker Registry Secret if it doesn't exist
       ========================= */
    stage('Create Docker Registry Secret') {
      steps {
        script {
          sh """
            kubectl get secret regcred -n ${params.ENV} || kubectl create secret docker-registry regcred -n ${params.ENV} \
              --docker-server=${REGISTRY} \
              --docker-username=${DOCKER_USERNAME} \
              --docker-password=${DOCKER_PASSWORD} 
          """
        }
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
        pwd
          ls -l frontend-hc
        sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' frontend-hc/frontendvalues_${params.ENV}.yaml
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
          sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' backend-hc/backendvalues_${params.ENV}.yaml
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
            git add frontend-hc/frontendvalues_${params.ENV}.yaml backend-hc/backendvalues_${params.ENV}.yaml
            git commit -m "Update images to tag ${IMAGE_TAG}" || echo "No changes"
            git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_9.git master
          """
        }
      }
    }

    /* =========================
       Apply Kubernetes and ArgoCD Resources (if not already applied)
       ========================= */
    stage('Apply Kubernetes & ArgoCD Resources') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY'] } }
      steps {
        script {
          // Check if ArgoCD resources already exist, if not apply them
          sh """
            kubectl get application backend -n argocd || kubectl apply -f argocd/backend_${params.ENV}.yaml
            kubectl get application database -n argocd || kubectl apply -f argocd/database-${params.ENV}.yaml
            kubectl get application frontend -n argocd || kubectl apply -f argocd/frontend_${params.ENV}.yaml
          """
          
          // Apply Kubernetes resources (PVC, PV, etc.)
          sh """
            kubectl get pvc shared-pvc -n ${params.ENV} || kubectl apply -f k8s/shared-pvc_${params.ENV}.yaml -n ${params.ENV}
            kubectl get pv shared-pv || kubectl apply -f k8s/shared-pv.yaml
          """
        }
      }
    }
  }
}
