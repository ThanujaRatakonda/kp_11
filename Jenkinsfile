
pipeline {
  agent any

  parameters {
    choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Target Kubernetes namespace/environment')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Image tag to use (default: BUILD_NUMBER)')
  }

  environment {
    REGISTRY = '10.131.103.92:8090'
    PROJECT  = 'kp_9'
    GIT_USER_EMAIL = 'ci-bot@example.com'
    GIT_USER_NAME  = 'ci-bot'
    // If IMAGE_TAG not provided, fallback to Jenkins BUILD_NUMBER
    FINAL_TAG = "${params.IMAGE_TAG ?: env.BUILD_NUMBER}"
  }

  options {
    timestamps()
    ansiColor('xterm')
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Docker Login (Harbor)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
          sh '''
            set -e
            echo "Logging into Harbor ${REGISTRY}..."
            docker login ${REGISTRY} -u "$HARBOR_USER" -p "$HARBOR_PASS"
          '''
        }
      }
    }

    stage('Build & Push Backend Image') {
      steps {
        sh '''
          set -e
          echo "Building backend image: ${REGISTRY}/${PROJECT}/backend:${FINAL_TAG}"
          docker build -t ${REGISTRY}/${PROJECT}/backend:${FINAL_TAG} backend/
          docker push ${REGISTRY}/${PROJECT}/backend:${FINAL_TAG}
        '''
      }
    }

    stage('Build & Push Frontend Image') {
      steps {
        sh '''
          set -e
          echo "Building frontend image: ${REGISTRY}/${PROJECT}/frontend:${FINAL_TAG}"
          docker build -t ${REGISTRY}/${PROJECT}/frontend:${FINAL_TAG} frontend/
          docker push ${REGISTRY}/${PROJECT}/frontend:${FINAL_TAG}
        '''
      }
    }

    stage('Prepare Kube Access') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            mkdir -p $HOME/.kube
            cp "$KUBECONFIG_FILE" $HOME/.kube/config
            chmod 600 $HOME/.kube/config

            echo "Kubernetes context:"
            kubectl config current-context
          '''
        }
      }
    }

    stage('Create/Ensure Namespace & Regcred') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
          sh '''
            set -e
            # Create namespace if not exists
            kubectl get ns ${ENV} >/dev/null 2>&1 || kubectl create ns ${ENV}

            # Create/Update docker-registry secret used by Helm charts (imagePullSecrets: regcred)
            kubectl -n ${ENV} create secret docker-registry regcred \
              --docker-server=${REGISTRY} \
              --docker-username="$HARBOR_USER" \
              --docker-password="$HARBOR_PASS" \
              --dry-run=client -o yaml | kubectl apply -f -
          '''
        }
      }
    }

    stage('Apply Base K8s (StorageClass, PV, PVC)') {
      steps {
        sh '''
          set -e
          # Substitute ${ENV} in k8s manifests and apply
          # If you have multiple files under k8s/, this applies all *.yaml/.yml
          for f in $(find k8s -type f \\( -name "*.yaml" -o -name "*.yml" \\)); do
            echo "Applying $f for ENV=${ENV}"
            ENV=${ENV} envsubst < "$f" | kubectl apply -f -
          done
        '''
      }
    }

    stage('Update Helm values with new image tags') {
      steps {
        sh '''
          set -e
          echo "Setting image tag to ${FINAL_TAG} in backend-hc/backendvalues.yaml and frontend-hc/frontendvalues.yaml"

          # Update backend tag
          sed -i 's|^  tag: ".*"|  tag: "'${FINAL_TAG}'"|' backend-hc/backendvalues.yaml

          # Update frontend tag
          sed -i 's|^  tag: ".*"|  tag: "'${FINAL_TAG}'"|' frontend-hc/frontendvalues.yaml

          echo "Preview changes:"
          grep -n 'tag: "' backend-hc/backendvalues.yaml
          grep -n 'tag: "' frontend-hc/frontendvalues.yaml
        '''
      }
    }

    stage('Commit & Push Changes (GitOps trigger)') {
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GHTOKEN')]) {
          sh '''
            set -e
            git config user.email "${GIT_USER_EMAIL}"
            git config user.name  "${GIT_USER_NAME}"

            git add backend-hc/backendvalues.yaml frontend-hc/frontendvalues.yaml
            git commit -m "CI: set image tag to ${FINAL_TAG} for ${ENV}"
            # Use token in remote URL
            git push https://$GHTOKEN@github.com/ThanujaRatakonda/kp_9.git HEAD:master
          '''
        }
      }
    }

    stage('Apply Argo CD Applications (ENV-subst)') {
      steps {
        sh '''
          set -e
          # The argocd/ file includes backend, database, frontend Applications.
          # Substitute ${ENV} and apply into argocd namespace.
          for f in $(find argocd -type f \\( -name "*.yaml" -o -name "*.yml" \\) -maxdepth 1); do
            echo "Applying Argo CD Application $f for ENV=${ENV}"
            ENV=${ENV} envsubst < "$f" | kubectl apply -n argocd -f -
          done

          echo "Argo CD Applications:"
          kubectl get applications.argoproj.io -n argocd
        '''
      }
    }

    stage('Wait for Deployment (basic checks)') {
      steps {
        sh '''
          set +e
          echo "Checking pods in namespace ${ENV}..."
          kubectl get pods -n ${ENV}

          echo "Services:"
          kubectl get svc -n ${ENV}

          echo "Ingresses:"
          kubectl get ingress -n ${ENV} || true

          echo "If frontend is NodePort, get NodePort:"
          kubectl get svc -n ${ENV} | awk '/frontend-hc/ {print}'
        '''
      }
    }

  } // stages

  post {
    success {
      echo "Pipeline completed: Images pushed, Helm values updated, Argo CD syncing to ${params.ENV}."
    }
    failure {
      echo "Pipeline failed. Check stage logs above."
    }
  }
}

