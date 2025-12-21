pipeline {
    agent any

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_9"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
        ARGOCD_SERVER = "10.131.103.92:8080"
        ARGOCD_TOKEN = credentials('argocd-token')
    }

    parameters {
        choice(
            name: 'ENV',
            choices: ['dev', 'qa', 'prod'],
            description: 'Select the target environment for deployment'
        )
        choice(
            name: 'ACTION',
            choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'],
            description: 'Choose FULL_PIPELINE, FRONTEND_ONLY, or BACKEND_ONLY'
        )
    }

    stages {
        stage('Checkout') {
            steps { git 'https://github.com/ThanujaRatakonda/kp_9.git' }
        }

        // Build & Scan Frontend
        stage('Build & Scan Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                sh "docker build -t frontend:${IMAGE_TAG} ./frontend"

                sh """
                    trivy image frontend:${IMAGE_TAG} \
                    --severity CRITICAL,HIGH \
                    --format json -o ${TRIVY_OUTPUT_JSON}
                """

                archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true

                script {
                    def vulnerabilities = sh(script: """
                        jq '[.Results[] |
                             (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                             (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                            ] | length' ${TRIVY_OUTPUT_JSON}
                    """, returnStdout: true).trim()

                    if (vulnerabilities.toInteger() > 0) {
                        error "CRITICAL/HIGH vulnerabilities found in frontend!"
                    }
                }
            }
        }

        stage('Push Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"
                        sh "docker tag frontend:${IMAGE_TAG} ${fullImage}"
                        sh "docker push ${fullImage}"
                        sh "docker rmi frontend:${IMAGE_TAG} || true"
                    }
                }
            }
        }

        // Build & Scan Backend
        stage('Build & Scan Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                sh "docker build -t backend:${IMAGE_TAG} ./backend"

                sh """
                    trivy image backend:${IMAGE_TAG} \
                    --severity CRITICAL,HIGH \
                    --format json -o ${TRIVY_OUTPUT_JSON}
                """

                archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true

                script {
                    def vulnerabilities = sh(script: """
                        jq '[.Results[] |
                             (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                             (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                            ] | length' ${TRIVY_OUTPUT_JSON}
                    """, returnStdout: true).trim()

                    if (vulnerabilities.toInteger() > 0) {
                        error "CRITICAL/HIGH vulnerabilities found in backend!"
                    }
                }
            }
        }

        stage('Push Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"
                        sh "docker tag backend:${IMAGE_TAG} ${fullImage}"
                        sh "docker push ${fullImage}"
                        sh "docker rmi backend:${IMAGE_TAG} || true"
                    }
                }
            }
        }

        // Apply Kubernetes manifests to selected environment
        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f k8s/ -n ${params.ENV}"
            }
        }

        // Trigger ArgoCD sync for selected environment
        stage('Trigger ArgoCD Sync') {
            steps {
                sh """
                    argocd app sync frontend --grpc-web --server ${ARGOCD_SERVER} --auth-token ${ARGOCD_TOKEN}
                    argocd app sync backend --grpc-web --server ${ARGOCD_SERVER} --auth-token ${ARGOCD_TOKEN}
                """
            }
        }
    }
}
