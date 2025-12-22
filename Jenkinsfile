pipeline {
  agent any

  environment {
    REGISTRY = "10.131.103.92:8090"
    PROJECT  = "kp_9"
    IMAGE_TAG = "${BUILD_NUMBER}"
    GIT_REPO = "https://github.com/ThanujaRatakonda/kp_9.git"
    DOCKER_USERNAME = "admin"
    DOCKER_PASSWORD = "Harbor12345"
    TRIVY_OUTPUT_JSON = "trivy-output.json"
  }

  parameters {
    choice(
      name: 'ACTION',
      choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'ARGOCD_ONLY'],
      description: 'Run full pipeline, only frontend/backend, or just apply ArgoCD resources'
    )
    choice(
      name: 'ENV',
      choices: ['dev', 'qa', 'BOTH'],
      description: 'Choose environment(s): dev/qa/BOTH'
    )
    booleanParam(
      name: 'SKIP_SCANNING',
      defaultValue: false,
      description: 'Skip Trivy vulnerability scanning'
    )
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'git-creds', url: "${GIT_REPO}", branch: 'master'
      }
    }

    /* =========================
       YOUR ORIGINAL WORKING SETUP (fixed for BOTH envs)
       ========================= */
    stage('Create Docker Registry Secret') {
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              kubectl get secret regcred -n ${ENV_NS} || kubectl create secret docker-registry regcred -n ${ENV_NS} \
                --docker-server=${REGISTRY} \
                --docker-username=${DOCKER_USERNAME} \
                --docker-password=${DOCKER_PASSWORD} 
              echo "✅ Registry secret for ${ENV_NS}"
            """
          }
        }
      }
    }

    stage('Apply Kubernetes Resources') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY'] } }
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              # Create namespace
              kubectl get namespace ${ENV_NS} || kubectl create namespace ${ENV_NS}
              
              # YOUR ORIGINAL STORAGE FILES
              kubectl get pvc shared-pvc -n ${ENV_NS} || kubectl apply -f k8s/shared-pvc.yaml -n ${ENV_NS}
              kubectl get pv shared-pv || kubectl apply -f k8s/shared-pv.yaml
              
              echo "✅ Storage ready for ${ENV_NS}"
            """
          }
        }
      }
    }

    /* =========================
       FRONTEND (Build → Trivy → Push)
       ========================= */
    stage('Build Frontend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        sh """
          docker build -t frontend:${IMAGE_TAG} ./frontend
          echo "✅ Frontend built"
        """
      }
    }

    stage('🛡️ Trivy Scan Frontend') {
      when { 
        expression { 
          params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] && 
          !params.SKIP_SCANNING
        } 
      }
      steps {
        sh """
          trivy image --severity CRITICAL,HIGH --format json -o ${TRIVY_OUTPUT_JSON} frontend:${IMAGE_TAG} \
          --skip-version-check
        """
        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true, allowEmptyArchive: true
        
        script {
          def vulnerabilities = sh(
            script: """
              jq '[.Results[] | 
                (.Vulnerabilities // [] | 
                  map(select(.Severity=="CRITICAL" or .Severity=="HIGH")) 
                ) 
              ] | length' ${TRIVY_OUTPUT_JSON}
            """, 
            returnStdout: true
          ).trim()
          
          echo "🔍 Frontend: ${vulnerabilities} CRITICAL/HIGH vulns"
          if (vulnerabilities.toInteger() > 0) {
            error "🚨 ${vulnerabilities} CRITICAL/HIGH vulns in frontend!"
          }
          echo "✅ Frontend PASSED!"
        }
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
            docker login ${REGISTRY} -u \$USER -p \$PASS
            docker tag frontend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
            docker rmi frontend:${IMAGE_TAG} || true
          """
        }
      }
    }

    stage('Update Frontend Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              test -f frontend-hc/frontendvalues_${ENV_NS}.yaml && \\
              sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' frontend-hc/frontendvalues_${ENV_NS}.yaml || \\
              sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' frontend-hc/frontendvalues.yaml
              echo "✅ Frontend values updated for ${ENV_NS}"
            """
          }
        }
      }
    }

    /* =========================
       BACKEND (Build → Trivy → Push)
       ========================= */
    stage('Build Backend Image') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        sh """
          docker build -t backend:${IMAGE_TAG} ./backend
          echo "✅ Backend built"
        """
      }
    }

    stage('🛡️ Trivy Scan Backend') {
      when { 
        expression { 
          params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] && 
          !params.SKIP_SCANNING
        } 
      }
      steps {
        sh """
          trivy image --severity CRITICAL,HIGH --format json -o ${TRIVY_OUTPUT_JSON} backend:${IMAGE_TAG} \
          --skip-version-check
        """
        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true, allowEmptyArchive: true
        
        script {
          def vulnerabilities = sh(
            script: """
              jq '[.Results[] | 
                (.Vulnerabilities // [] | 
                  map(select(.Severity=="CRITICAL" or .Severity=="HIGH")) 
                ) 
              ] | length' ${TRIVY_OUTPUT_JSON}
            """, 
            returnStdout: true
          ).trim()
          
          echo "🔍 Backend: ${vulnerabilities} CRITICAL/HIGH vulns"
          if (vulnerabilities.toInteger() > 0) {
            error "🚨 ${vulnerabilities} CRITICAL/HIGH vulns in backend!"
          }
          echo "✅ Backend PASSED!"
        }
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
            docker login ${REGISTRY} -u \$USER -p \$PASS
            docker tag backend:${IMAGE_TAG} ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
            docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
            docker rmi backend:${IMAGE_TAG} || true
          """
        }
      }
    }

    stage('Update Backend Helm Values') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              test -f backend-hc/backendvalues_${ENV_NS}.yaml && \\
              sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' backend-hc/backendvalues_${ENV_NS}.yaml || \\
              sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' backend-hc/backendvalues.yaml
              echo "✅ Backend values updated for ${ENV_NS}"
            """
          }
        }
      }
    }

    /* =========================
       COMMIT & ARGO CD
       ========================= */
    stage('Commit & Push Helm Changes') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'GitHub',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          sh """
            git config user.name "thanuja"
            git config user.email "ratakondathanuja@gmail.com"
            git add frontend-hc/*.yaml backend-hc/*.yaml
            git commit -m "Update images to tag ${IMAGE_TAG} for ${params.ENV}" || echo "No changes"
            git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_9.git master
          """
        }
      }
    }

    stage('Apply ArgoCD Resources') {
      when { expression { params.ACTION in ['FULL_PIPELINE', 'ARGOCD_ONLY'] } }
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              kubectl get application backend -n argocd || kubectl apply -f argocd/backend-app.yaml
              kubectl get application frontend -n argocd || kubectl apply -f argocd/frontend-app.yaml
              kubectl get application database -n argocd || kubectl apply -f argocd/database-app.yaml
              
              # Hard refresh
              kubectl annotate application backend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application frontend -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
              kubectl annotate application database -n argocd argocd.argoproj.io/refresh=hard --overwrite || true
            """
          }
        }
      }
    }

    stage('✅ Verify') {
      steps {
        script {
          def envs = params.ENV == 'BOTH' ? ['dev', 'qa'] : [params.ENV]
          for (def ENV_NS : envs) {
            sh """
              echo "=== ${ENV_NS} ==="
              kubectl get pods -n ${ENV_NS}
              kubectl get svc -n ${ENV_NS}
            """
          }
        }
      }
    }
  }

  post {
    always {
      sh 'docker logout ${REGISTRY} || true'
      sh 'docker image prune -f || true'
      archiveArtifacts artifacts: 'trivy-output.json', allowEmptyArchive: true
    }
  }
}
