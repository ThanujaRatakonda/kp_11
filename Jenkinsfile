
pipeline {
  agent any

  parameters {
    choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Target K8s namespace/environment')
  }

  environment {
    REGISTRY = '10.131.103.92:8090'
    PROJECT  = 'kp_9'
    IMAGE_TAG = "${env.BUILD_NUMBER}"       // use Jenkins build number as tag
    SCAN_SEVERITY = 'HIGH,CRITICAL'         // Trivy severity gate
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
            docker login ${REGISTRY} -u "$HARBOR_USER" -p "$HARBOR_PASS"
          '''
        }
      }
    }

    /* ===========================
       Backend: Build → Scan → Push
       =========================== */
    stage('Build Backend Image') {
      steps {
        sh '''
          set -e
          echo "Building backend: ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}"
          docker build -t ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} backend/
        '''
      }
    }

    stage('Trivy Scan Backend') {
      steps {
        sh '''
          set -e
          mkdir -p reports

          echo "Trivy (table report) backend..."
          trivy image --severity ${SCAN_SEVERITY} --ignore-unfixed \
            --format table -o reports/backend_${IMAGE_TAG}_trivy.txt \
            ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}

          echo "Trivy (SARIF) backend..."
          trivy image --severity ${SCAN_SEVERITY} --ignore-unfixed \
            --format sarif -o reports/backend_${IMAGE_TAG}_trivy.sarif \
            ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}

          echo "Trivy gate: fail on ${SCAN_SEVERITY}..."
          trivy image --severity ${SCAN_SEVERITY} --ignore-unfixed \
            --exit-code 1 \
            ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
        '''
      }
    }

    stage('Push Backend Image') {
      steps {
        sh '''
          set -e
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
        '''
      }
    }

    /* ===========================
       Frontend: Build → Scan → Push
       =========================== */
    stage('Build Frontend Image') {
      steps {
        sh '''
          set -e
          echo "Building frontend: ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}"
          docker build -t ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} frontend/
        '''
      }
    }

    stage('Trivy Scan Frontend') {
      steps {
        sh '''
          set -e
          mkdir -p reports

          echo "Trivy (table report) frontend..."
          trivy image --severity ${SCAN_SEVERITY} --ignore-unfixed \
            --format table -o reports/frontend_${IMAGE_TAG}_trivy.txt \
            ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}

          echo "Trivy (SARIF) frontend..."
          trivy image --severity ${SCAN_SEVERITY} --ignore-unfixed \
            --format sarif -o reports/frontend_${IMAGE_TAG}_trivy.sarif \
            ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}

          echo "Trivy gate: fail on ${SCAN_SEVERITY}..."
          trivy image --severity ${SCAN_SEVERITY} --ignore-unfixed \
            --exit-code 1 \
            ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
        '''
      }
    }

    stage('Push Frontend Image') {
      steps {
        sh '''
          set -e
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
        '''
      }
    }

    /* ===========================
       K8s Base: Namespace, regcred, storage
       =========================== */
    stage('Apply K8s Base & regcred') {
      steps {
        withCredentials([
          file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE'),
          usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')
        ]) {
          sh '''
            set -e
            export KUBECONFIG=$KUBECONFIG_FILE

            # Ensure namespace exists
            kubectl get ns ${ENV} >/dev/null 2>&1 || kubectl create ns ${ENV}

            # Create/Update imagePullSecrets: regcred
            kubectl -n ${ENV} create secret docker-registry regcred \
              --docker-server=${REGISTRY} \
              --docker-username="$HARBOR_USER" \
              --docker-password="$HARBOR_PASS" \
              --docker-email="ci@example.com" \
              --dry-run=client -o yaml | kubectl apply -f -

            # Apply StorageClass, PV, PVC with ENV substitution
            for f in $(ls k8s/*.yaml k8s/*.yml 2>/dev/null); do
              echo "Applying ${f} for ENV=${ENV}"
              ENV=${ENV} envsubst < "$f" | kubectl apply -f -
            done
          '''
        }
      }
    }

    /* ===========================
       GitOps: bump Helm image tags and push to GitHub
       =========================== */
    stage('Update Helm Values & Push (GitOps)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GH_USER', passwordVariable: 'GH_PAT')]) {
          sh '''
            set -e

            # Bump image tags in Helm values
            sed -i 's|^  tag: ".*"|  tag: "'${IMAGE_TAG}'"|' backend-hc/backendvalues.yaml
            sed -i 's|^  tag: ".*"|  tag: "'${IMAGE_TAG}'"|' frontend-hc/frontendvalues.yaml

            git config user.email "ci@example.com"
            git config user.name "ci-bot"
            git add backend-hc/backendvalues.yaml frontend-hc/frontendvalues.yaml
            git commit -m "CI: set image tag ${IMAGE_TAG} for ${ENV}" || echo "No changes to commit"

            # Push using GitHub username + PAT
            git push https://${GH_USER}:${GH_PAT}@github.com/ThanujaRatakonda/kp_9.git HEAD:master
          '''
        }
      }
    }

    /* ===========================
       Argo CD: apply Applications with ENV subs
       =========================== */
    stage('Apply Argo CD Applications') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG=$KUBECONFIG_FILE

            for f in $(ls argocd/*.yaml argocd/*.yml 2>/dev/null); do
              echo "Applying Argo CD app: ${f}"
              ENV=${ENV} envsubst < "$f" | kubectl apply -n argocd -f -
            done

            kubectl get applications.argoproj.io -n argocd || true
          '''
        }
      }
    }

    /* ===========================
       Quick checks
       =========================== */
    stage('Quick Check') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set +e
            export KUBECONFIG=$KUBECONFIG_FILE

            echo "Pods in ${ENV}:"
            kubectl get pods -n ${ENV}

            echo "Services in ${ENV}:"
            kubectl get svc -n ${ENV}

            echo "Ingress in ${ENV}:"
            kubectl get ingress -n ${ENV} || true
          '''
        }
      }
    }

  } // stages

  post {
    always {
      archiveArtifacts artifacts: 'reports/**/*', onlyIfSuccessful: false, allowEmptyArchive: true
    }
    success {
      echo "✅ Done: Images built, scanned, pushed; Helm values updated; Argo CD syncing to ${params.ENV}."
    }
    failure {
      echo "❌ Pipeline failed. See stage logs. Trivy gates on ${SCAN_SEVERITY} may have blocked push."
    }
  }
}
