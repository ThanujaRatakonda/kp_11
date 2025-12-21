
pipeline {
  agent any

  environment {
    IMAGE_TAG       = "${env.BUILD_NUMBER}"
    HARBOR_URL      = "10.131.103.92:8090"
    HARBOR_PROJECT  = "kp_6"                 // your Harbor project
    TRIVY_SEVERITY  = "CRITICAL,HIGH"
    TRIVY_IGNORE_UNFIXED = "true"            // set to "false" if you want unfixed counted
  }

  parameters {
    choice(
      name: 'ACTION',
      choices: ['FULL_PIPELINE', 'SCALE_ONLY', 'FRONTEND_ONLY', 'BACKEND_ONLY'],
      description: 'Choose FULL_PIPELINE, SCALE_ONLY, FRONTEND_ONLY, or BACKEND_ONLY'
    )
    string(name: 'NAMESPACE', defaultValue: 'dev', description: 'Target Kubernetes namespace')
    string(name: 'FRONTEND_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for frontend')
    string(name: 'BACKEND_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for backend')
    string(name: 'DB_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for database')
  }

  stages {

    stage('Checkout') {
      when { expression { params.ACTION != 'SCALE_ONLY' } }
      steps {
        // Use your repo that has k8s/ frontend/ backend/
        git 'https://github.com/ThanujaRatakonda/kp_6.git'
      }
    }

    /* ====== Login to Harbor ====== */
    stage('Docker Login (Harbor)') {
      when { expression { params.ACTION != 'SCALE_ONLY' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
          sh '''
            set -e
            echo "$HARBOR_PASS" | docker login ${HARBOR_URL} -u "$HARBOR_USER" --password-stdin
          '''
        }
      }
    }

    /* ====== Ensure K8s access + namespace + regcred ====== */
    stage('Prepare K8s Namespace & regcred') {
      steps {
        withCredentials([
          file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE'),
          usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')
        ]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            # Ensure namespace exists
            kubectl get ns ${NAMESPACE} >/dev/null 2>&1 || kubectl create ns ${NAMESPACE}

            # Create/Update docker-registry secret (imagePullSecrets: regcred)
            kubectl -n ${NAMESPACE} create secret docker-registry regcred \
              --docker-server=${HARBOR_URL} \
              --docker-username="$HARBOR_USER" \
              --docker-password="$HARBOR_PASS" \
              --docker-email="ci@example.com" \
              --dry-run=client -o yaml | kubectl apply -f -
          '''
        }
      }
    }

    /* ====== Storage base (SC/PV/PVC) ====== */
    stage('Setup Storage') {
      when { expression { params.ACTION == 'FULL_PIPELINE' } }
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            # Apply all; -n is ignored for cluster-scoped (SC/PV) and used for PVC
            kubectl -n ${NAMESPACE} apply -f k8s/shared-storage-class.yaml || true
            kubectl -n ${NAMESPACE} apply -f k8s/shared-pv.yaml || true
            kubectl -n ${NAMESPACE} apply -f k8s/shared-pvc.yaml || true
          '''
        }
      }
    }

    /* ====== Frontend: Build → Scan → Push → Deploy ====== */
    stage('Build Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh 'docker build -t frontend:${IMAGE_TAG} ./frontend'
      }
    }

    stage('Scan Frontend (Trivy)') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh '''
          set -e
          mkdir -p reports

          IGNORE_FLAG=""
          if [ "${TRIVY_IGNORE_UNFIXED}" = "true" ]; then
            IGNORE_FLAG="--ignore-unfixed"
          fi

          # table + json + sarif
          trivy image frontend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format table -o reports/frontend_${IMAGE_TAG}_trivy.txt

          trivy image frontend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format json  -o reports/frontend_${IMAGE_TAG}_trivy.json

          trivy image frontend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format sarif -o reports/frontend_${IMAGE_TAG}_trivy.sarif

          # Gate on HIGH/CRITICAL using jq over JSON
          VULN_COUNT=$(jq '[.Results[]? | (.Vulnerabilities // [])[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")] | length' \
            reports/frontend_${IMAGE_TAG}_trivy.json)

          echo "Frontend CRITICAL/HIGH vuln count: ${VULN_COUNT}"
          if [ "$VULN_COUNT" -gt 0 ]; then
            echo "Blocking deploy: CRITICAL/HIGH vulnerabilities found in frontend."
            exit 1
          fi
        '''
        archiveArtifacts artifacts: "reports/frontend_${IMAGE_TAG}_trivy.*", fingerprint: true, allowEmptyArchive: false
      }
    }

    stage('Push Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        script {
          def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
          sh """
            set -e
            docker tag frontend:${IMAGE_TAG} ${fullImage}
            docker push ${fullImage}
            docker rmi frontend:${IMAGE_TAG} || true
          """
        }
      }
    }

    stage('Deploy Frontend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            # Render a temp file with ${IMAGE_TAG} replaced, then apply to the target namespace
            TMPF=$(mktemp)
            sed 's|__IMAGE_TAG__|'"${IMAGE_TAG}"'|g' k8s/frontend-deployment.yaml > "$TMPF"
            kubectl -n ${NAMESPACE} apply -f "$TMPF"
            rm -f "$TMPF"
          '''
        }
      }
    }

    /* ====== Backend: Build → Scan → Push → Deploy ====== */
    stage('Build Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh 'docker build -t backend:${IMAGE_TAG} ./backend'
      }
    }

    stage('Scan Backend (Trivy)') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh '''
          set -e
          mkdir -p reports

          IGNORE_FLAG=""
          if [ "${TRIVY_IGNORE_UNFIXED}" = "true" ]; then
            IGNORE_FLAG="--ignore-unfixed"
          fi

          trivy image backend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format table -o reports/backend_${IMAGE_TAG}_trivy.txt

          trivy image backend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format json  -o reports/backend_${IMAGE_TAG}_trivy.json

          trivy image backend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format sarif -o reports/backend_${IMAGE_TAG}_trivy.sarif

          VULN_COUNT=$(jq '[.Results[]? | (.Vulnerabilities // [])[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")] | length' \
            reports/backend_${IMAGE_TAG}_trivy.json)

          echo "Backend CRITICAL/HIGH vuln count: ${VULN_COUNT}"
          if [ "$VULN_COUNT" -gt 0 ]; then
            echo "Blocking deploy: CRITICAL/HIGH vulnerabilities found in backend."
            exit 1
          fi
        '''
        archiveArtifacts artifacts: "reports/backend_${IMAGE_TAG}_trivy.*", fingerprint: true, allowEmptyArchive: false
      }
    }

    stage('Push Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        script {
          def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
          sh """
            set -e
            docker tag backend:${IMAGE_TAG} ${fullImage}
            docker push ${fullImage}
            docker rmi backend:${IMAGE_TAG} || true
          """
        }
      }
    }

    stage('Deploy Backend + Ingress') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            TMPB=$(mktemp)
            sed 's|__IMAGE_TAG__|'"${IMAGE_TAG}"'|g' k8s/backend-deployment.yaml > "$TMPB"
            kubectl -n ${NAMESPACE} apply -f "$TMPB"
            rm -f "$TMPB"

            # Ingress (optional file)
            kubectl -n ${NAMESPACE} apply -f k8s/backend-ingress.yaml || true
          '''
        }
      }
    }

    /* ====== Database ====== */
    stage('Deploy Database') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'SCALE_ONLY'] } }
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            kubectl -n ${NAMESPACE} apply -f k8s/database-deployment.yaml || true
            kubectl -n ${NAMESPACE} scale statefulset database --replicas=${DB_REPLICA_COUNT}
          '''
        }
      }
    }

    /* ====== Scaling ====== */
    stage('Scale Frontend & Backend') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'SCALE_ONLY', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            if [ "${ACTION}" = "FULL_PIPELINE" ] || [ "${ACTION}" = "SCALE_ONLY" ] || [ "${ACTION}" = "FRONTEND_ONLY" ]; then
              kubectl -n ${NAMESPACE} scale deployment frontend --replicas=${FRONTEND_REPLICA_COUNT}
            fi

            if [ "${ACTION}" = "FULL_PIPELINE" ] || [ "${ACTION}" = "SCALE_ONLY" ] || [ "${ACTION}" = "BACKEND_ONLY" ]; then
              kubectl -n ${NAMESPACE} scale deployment backend --replicas=${BACKEND_REPLICA_COUNT}
            fi

            kubectl -n ${NAMESPACE} get deployments
          '''
        }
      }
    }

    /* ====== VPA (optional) ====== */
    stage('Deploy VPA (optional)') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'SCALE_ONLY', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            kubectl -n ${NAMESPACE} apply -f k8s/frontend-vpa.yaml || true
            kubectl -n ${NAMESPACE} apply -f k8s/backend-vpa.yaml || true
          '''
        }
      }
    }

    /* ====== Quick check ====== */
    stage('Quick Check') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set +e
            export KUBECONFIG="$KUBECONFIG_FILE"

            echo "Pods in ${NAMESPACE}:"
            kubectl -n ${NAMESPACE} get pods -o wide

            echo "Services in ${NAMESPACE}:"
            kubectl -n ${NAMESPACE} get svc

            echo "Ingresses in ${NAMESPACE}:"
            kubectl -n ${NAMESPACE} get ingress || true
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
      echo "✅ Done: Built, scanned, pushed; applied to namespace ${params.NAMESPACE}. Trivy reports archived."
    }
    failure {
      echo "❌ Failed. Check Trivy gates (HIGH/CRITICAL), Harbor login, or Kubernetes apply logs."
    }
  }
}

