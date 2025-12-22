pipeline {
    agent any

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_6"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
        ENV = "dev"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ThanujaRatakonda/kp_6.git'
            }
        }

        // ---------- FRONTEND ----------
        stage('Build Frontend') {
            steps {
                sh "docker build -t frontend:${IMAGE_TAG} ./frontend"
            }
        }

        stage('Scan Frontend') {
            steps {
                sh """
                trivy image frontend:${IMAGE_TAG} \
                --severity CRITICAL,HIGH \
                --format json -o ${TRIVY_OUTPUT_JSON}
                """
                archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true
            }
        }

        stage('Push Frontend') {
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(
                        credentialsId: 'harbor-creds',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh """
                        echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin
                        docker tag frontend:${IMAGE_TAG} ${fullImage}
                        docker push ${fullImage}
                        docker rmi frontend:${IMAGE_TAG} || true
                        """
                    }
                }
            }
        }

        // ---------- BACKEND ----------
        stage('Build Backend') {
            steps {
                sh "docker build -t backend:${IMAGE_TAG} ./backend"
            }
        }

        stage('Scan Backend') {
            steps {
                sh """
                trivy image backend:${IMAGE_TAG} \
                --severity CRITICAL,HIGH \
                --format json -o ${TRIVY_OUTPUT_JSON}
                """
                archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true
            }
        }

        stage('Push Backend') {
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(
                        credentialsId: 'harbor-creds',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh """
                        echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin
                        docker tag backend:${IMAGE_TAG} ${fullImage}
                        docker push ${fullImage}
                        docker rmi backend:${IMAGE_TAG} || true
                        """
                    }
                }
            }
        }

        // ---------- KUBERNETES APPLY ----------
        stage('Apply Kubernetes Manifests') {
            steps {
                sh "kubectl apply -f k8s/ -n ${ENV}"
                sh "kubectl apply -f argocd/ -n ${ENV}"
            }
        }
    }
}
