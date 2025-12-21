pipeline {
    agent any
    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_7"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
    }
    parameters {
        choice(
            name: 'ACTION',
            choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'],
            description: 'Select pipeline action'
        )
    }

    stages {
        stage('Checkout') {
            when { expression { params.ACTION != 'SCALE_ONLY' } }
            steps { git 'https://github.com/ThanujaRatakonda/kp_7.git' }
        }

        stage('Build Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps { sh "docker build -t frontend:${env.IMAGE_TAG} ./frontend" }
        }

        stage('Scan Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                sh """
                    trivy image frontend:${env.IMAGE_TAG} \
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
                    if (vulnerabilities.toInteger() > 0) { error "CRITICAL/HIGH vulnerabilities found in frontend!" }
                }
            }
        }

        stage('Push Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${env.IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh """
                            echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin
                            docker tag frontend:${env.IMAGE_TAG} ${fullImage}
                            docker push ${fullImage}
                            docker rmi frontend:${env.IMAGE_TAG} || true
                        """
                    }
                }
            }
        }

        stage('Build Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps { sh "docker build -t backend:${env.IMAGE_TAG} ./backend" }
        }

        stage('Scan Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                sh """
                    trivy image backend:${env.IMAGE_TAG} \
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
                    if (vulnerabilities.toInteger() > 0) { error "CRITICAL/HIGH vulnerabilities found in backend!" }
                }
            }
        }

        stage('Push Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/backend:${env.IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh """
                            echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin
                            docker tag backend:${env.IMAGE_TAG} ${fullImage}
                            docker push ${fullImage}
                            docker rmi backend:${env.IMAGE_TAG} || true
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f k8s/ -n dev"
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                sh "argocd app sync kp_7 --prune"
            }
        }
    }

    post {
        always {
            node {
                echo "Cleaning up Docker images..."
                sh "docker rmi frontend:${env.IMAGE_TAG} backend:${env.IMAGE_TAG} || true"
            }
        }
        failure {
            echo "Pipeline failed. Check Trivy scan or deployment logs."
        }
    }
}
