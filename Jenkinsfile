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
            name: 'SERVICE',
            choices: ['all', 'frontend', 'backend'],
            description: 'Which service to build and deploy'
        )
        choice(
            name: 'ENV',
            choices: ['dev', 'qa', 'prod'],
            description: 'Environment namespace'
        )
    }

    stages {

        stage('Checkout') {
            steps { git 'https://github.com/ThanujaRatakonda/kp_7.git' }
        }

        stage('Build & Push Docker Images') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        if (params.SERVICE in ['all', 'frontend']) {
                            sh "docker build -t frontend:${IMAGE_TAG} frontend/"
                            sh "docker tag frontend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
                            sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"
                            sh "docker push ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
                            sh "docker rmi frontend:${IMAGE_TAG} || true"
                        }
                        if (params.SERVICE in ['all', 'backend']) {
                            sh "docker build -t backend:${IMAGE_TAG} backend/"
                            sh "docker tag backend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
                            sh "docker push ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
                            sh "docker rmi backend:${IMAGE_TAG} || true"
                        }
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    if (params.SERVICE in ['all', 'frontend']) {
                        sh """
                            trivy image ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG} \
                            --severity CRITICAL,HIGH \
                            --format json -o ${TRIVY_OUTPUT_JSON}
                        """
                        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true
                        def vuln = sh(script: """
                            jq '[.Results[] | 
                                 (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                                 (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                                ] | length' ${TRIVY_OUTPUT_JSON}
                        """, returnStdout: true).trim()
                        if (vuln.toInteger() > 0) { error "CRITICAL/HIGH vulnerabilities found in frontend!" }
                    }
                    if (params.SERVICE in ['all', 'backend']) {
                        sh """
                            trivy image ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG} \
                            --severity CRITICAL,HIGH \
                            --format json -o ${TRIVY_OUTPUT_JSON}
                        """
                        archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true
                        def vuln = sh(script: """
                            jq '[.Results[] | 
                                 (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                                 (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                                ] | length' ${TRIVY_OUTPUT_JSON}
                        """, returnStdout: true).trim()
                        if (vuln.toInteger() > 0) { error "CRITICAL/HIGH vulnerabilities found in backend!" }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f k8s/ -n ${params.ENV}"
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                script {
                    def apps = ['frontend', 'backend']
                    apps.each { app ->
                        if (params.SERVICE in ['all', app]) {
                            sh """
                                argocd app sync ${app} \
                                --grpc-web \
                                --server <ARGOCD_SERVER> \
                                --auth-token <ARGOCD_AUTH_TOKEN>
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up Docker images..."
            sh "docker rmi frontend:${IMAGE_TAG} backend:${IMAGE_TAG} || true"
        }
        failure {
            echo "Pipeline failed. Check Trivy scan or deployment logs."
        }
        success {
            echo "Pipeline succeeded! ${params.SERVICE} deployed to ${params.ENV}"
        }
    }
}
