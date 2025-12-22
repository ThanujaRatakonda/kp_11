pipeline {
  agent any

  environment {
    ENV = "dev"

    HARBOR_URL = "10.131.103.92:8090"
    HARBOR_PROJECT = "kp_9"

    IMAGE_TAG = "${BUILD_NUMBER}"
    TRIVY_OUTPUT_JSON = "trivy-output.json"
  }

  stages {

// CHECKOUT
      
    stage('Checkout') {
      steps {
        git branch: 'master',
            url: 'https://github.com/ThanujaRatakonda/kp_9.git'
      }
    }


    //   BACKEND

    stage('Build Backend Image') {
      steps {
        sh "docker build -t backend:${IMAGE_TAG} ./backend"
      }
    }

    stage('Scan Backend Image') {
      steps {
        sh """
          trivy image backend:${IMAGE_TAG} \
          --severity CRITICAL,HIGH \
          --format json \
          -o ${TRIVY_OUTPUT_JSON}
        """

        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true

        script {
          def count = sh(
            script: "jq '[.Results[].Vulnerabilities[]? | select(.Severity==\"CRITICAL\" or .Severity==\"HIGH\")] | length' ${TRIVY_OUTPUT_JSON}",
            returnStdout: true
          ).trim()

          if (count.toInteger() > 0) {
            error "CRITICAL/HIGH vulnerabilities found in BACKEND "
          }
        }
      }
    }

    stage('Push Backend Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'harbor-creds',
          usernameVariable: 'HARBOR_USER',
          passwordVariable: 'HARBOR_PASS'
        )]) {
          sh """
            docker login ${HARBOR_URL} -u $HARBOR_USER -p $HARBOR_PASS
            docker tag backend:${IMAGE_TAG} \
              ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
            docker push \
              ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Update Backend Helm Values') {
      steps {
        sh """
          sed -i 's/tag: \".*\"/tag: \"${IMAGE_TAG}\"/' \
          backend-hc/backendvalues.yaml
        """
      }
    }

    //   FRONTEND

    stage('Build Frontend Image') {
      steps {
        sh "docker build -t frontend:${IMAGE_TAG} ./frontend"
      }
    }

    stage('Scan Frontend Image') {
      steps {
        sh """
          trivy image frontend:${IMAGE_TAG} \
          --severity CRITICAL,HIGH \
          --format json \
          -o ${TRIVY_OUTPUT_JSON}
        """

        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true

        script {
          def count = sh(
            script: "jq '[.Results[].Vulnerabilities[]? | select(.Severity==\"CRITICAL\" or .Severity==\"HIGH\")] | length' ${TRIVY_OUTPUT_JSON}",
            returnStdout: true
          ).trim()

          if (count.toInteger() > 0) {
            error "CRITICAL/HIGH vulnerabilities found in FRONTEND "
          }
        }
      }
    }

    stage('Push Frontend Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'harbor-creds',
          usernameVariable: 'HARBOR_USER',
          passwordVariable: 'HARBOR_PASS'
        )]) {
          sh """
            docker login ${HARBOR_URL} -u $HARBOR_USER -p $HARBOR_PASS
            docker tag frontend:${IMAGE_TAG} \
              ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
            docker push \
              ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Update Frontend Helm Values') {
      steps {
        sh """
          sed -i 's/tag: \".*\"/tag: \"${IMAGE_TAG}\"/' \
          frontend-hc/frontendvalues.yaml
        """
      }
    }


     //  GITOPS COMMIT

    stage('Commit & Push Helm Changes') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'git-creds',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_PASS'
        )]) {
          sh """
            git config user.name "thanuja"
            git config user.email "ratakondathanuja@gmail.com"

            git add .
            git commit -m "ci(${ENV}): update images to ${IMAGE_TAG}"
            git push https://${GIT_USER}:${GIT_PASS}@github.com/ThanujaRatakonda/kp_9.git master
          """
        }
      }
    }

  }
}
