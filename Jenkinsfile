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

      // Render templates and apply correctly
      sh """
        set -euo pipefail

        export ENV="${params.ENV}"

        # Prepare rendered directories
        rm -rf rendered
        mkdir -p rendered/k8s rendered/argocd

        # Render all k8s YAMLs (handles any ${ENV} in files)
        for f in k8s/*.yaml; do
          envsubst < "$f" > "rendered/k8s/$(basename "$f")"
        done

        # Render ArgoCD YAMLs if they reference ${ENV}
        for f in argocd/*.yaml; do
          envsubst < "$f" > "rendered/argocd/$(basename "$f")"
        done

        echo "==== Rendered Namespace ===="
        cat rendered/k8s/namespace.yaml || true

        # 1) Apply Namespace (cluster-scoped) — DO NOT use -n
        kubectl apply -f rendered/k8s/namespace.yaml

        # 2) Apply other k8s resources into the target namespace
        #    Exclude namespace.yaml to avoid re-applying it
        find rendered/k8s -maxdepth 1 -type f -name '*.yaml' ! -name 'namespace.yaml' -print0 \
          | xargs -0 -I{} kubectl apply -f {} -n "${ENV}"

        # 3) Apply ArgoCD apps in argocd namespace (if present)
        if [ -d rendered/argocd ]; then
          kubectl apply -f rendered/argocd/ -n argocd
        fi
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
