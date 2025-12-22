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
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'],
      description: 'Run full pipeline or only frontend/backend'
    )
    // Define the environment parameter for selecting the namespace dynamically
    choice(
      name: 'ENV',
      choices: ['dev', 'qa', 'staging', 'prod'],
      description: 'Select the target environment (namespace)'
    )
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
      when { expression { params.ACTION in ['FULL_PIPELINE','FRONTEND_ONLY'] } }
      steps {
        sh """
          docker build -t frontend:${IMAGE_TAG} ./frontend
        """
      }
    }

    stage('Push Frontend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE','FRONTEND_ONLY'] } }
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
      when { expression { params.ACTION in ['FULL_PIPELINE','FRONTEND_ONLY'] } }
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
      when { expression { params.ACTION in ['FULL_PIPELINE','BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t backend:${IMAGE_TAG} ./backend
        """
      }
    }

    stage('Push Backend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE','BACKEND_ONLY'] } }
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
      when { expression { params.ACTION in ['FULL_PIPELINE','BACKEND_ONLY'] } }
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
       APPLY K8S RESOURCES (Manifests)
       ========================= */
  stage('Apply Kubernetes & ArgoCD Resources') {
  steps {
    script {
      echo "Applying resources for namespace: ${params.ENV}"

      // Set the environment variable explicitly for envsubst to use
      sh """
        export ENV=${params.ENV}
        echo "Substituting ENV in namespace.yaml"
        envsubst < k8s/namespace.yaml > k8s/namespace_tmp.yaml
        kubectl apply -f k8s/namespace_tmp.yaml  # Apply namespace

        # Apply PV and PVC
        kubectl apply -f k8s/shared-pv.yaml
        kubectl apply -f k8s/shared-pvc.yaml --namespace=${params.ENV}
        kubectl apply -f k8s/shared-storage-class.yaml  # Apply storage class
      """

      // Apply remaining Kubernetes resources with the correct namespace
      sh """
        kubectl apply -f k8s/ --namespace=${params.ENV}
      """

      // Apply ArgoCD resources separately
      sh """
        kubectl apply -f argocd/ --namespace=argocd
      """
    }
  }
}

  }

  post {
    success {
      echo "Argo CD will deploy automatically."
    }
  }
}
