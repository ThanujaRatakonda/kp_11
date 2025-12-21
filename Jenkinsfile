pipeline {
    agent any
    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_6"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
        ENV = "dev" // Change dynamically if needed
    }
    parameters {
        choice(
            name: 'ACTION',
            choices: ['FULL_PIPELINE', 'SCALE_ONLY', 'FRONTEND_ONLY', 'BACKEND_ONLY'],
            description: 'Choose FULL_PIPELINE, SCALE_ONLY, FRONTEND_ONLY, or BACKEND_ONLY'
        )
        string(name: 'FRONTEND_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for frontend')
        string(name: 'BACKEND_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for backend')
        string(name: 'DB_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for database')
    }

    stages {
        stage('Checkout') {
            when { expression {  params.ACTION != 'SCALE_ONLY' } }
            steps { git 'https://github.com/ThanujaRatakonda/kp_6.git' }
        }

        stage('Setup Storage') {
            when { expression { params.ACTION == 'FULL_PIPELINE' } }
            steps {
                sh "kubectl apply -f k8s/shared-storage-class.yaml || true"
                sh "kubectl apply -f k8s/shared-pv.yaml || true"
                sh "kubectl apply -f k8s/shared-pvc.yaml || true"
            }
        }

        // --- FRONTEND ---
        stage('Build Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps { sh "docker build -t frontend:${IMAGE_TAG} ./frontend" }
        }

        stage('Scan Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                sh """
                    trivy image frontend:${IMAGE_TAG} --severity CRITICAL,HIGH --format json -o ${TRIVY_OUTPUT_JSON}
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

        stage('Deploy Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                sh "sed -i 's/__IMAGE_TAG__/${IMAGE_TAG}/g' k8s/frontend-deployment.yaml"
                sh "kubectl apply -f k8s/frontend-deployment.yaml"
            }
        }

        // --- BACKEND ---
        stage('Build Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps { sh "docker build -t backend:${IMAGE_TAG} ./backend" }
        }

        stage('Scan Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                sh """
                    trivy image backend:${IMAGE_TAG} --severity CRITICAL,HIGH --format json -o ${TRIVY_OUTPUT_JSON}
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

        stage('Deploy Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                sh "sed -i 's/__IMAGE_TAG__/${IMAGE_TAG}/g' k8s/backend-deployment.yaml"
                sh "kubectl apply -f k8s/backend-deployment.yaml"
                sh "kubectl apply -f k8s/backend-ingress.yaml || true"
            }
        }

        // --- DATABASE ---
        stage('Deploy Database') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'SCALE_ONLY'] } }
            steps {
                sh "kubectl apply -f k8s/database-deployment.yaml || true"
                sh "kubectl scale statefulset database --replicas=${params.DB_REPLICA_COUNT}"
            }
        }

        // --- SCALING ---
        stage('Scale Frontend & Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'SCALE_ONLY', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
            steps {
                script {
                    if (params.ACTION in ['FULL_PIPELINE', 'SCALE_ONLY', 'FRONTEND_ONLY']) {
                        sh "kubectl scale deployment frontend --replicas=${params.FRONTEND_REPLICA_COUNT}"
                    }
                    if (params.ACTION in ['FULL_PIPELINE', 'SCALE_ONLY', 'BACKEND_ONLY']) {
                        sh "kubectl scale deployment backend --replicas=${params.BACKEND_REPLICA_COUNT}"
                    }
                    sh "kubectl get deployments"
                }
            }
        }

        stage('Deploy HPA/VPA') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'SCALE_ONLY', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
            steps {
                sh "kubectl apply -f k8s/frontend-vpa.yaml || true"
                sh "kubectl apply -f k8s/backend-vpa.yaml || true"
            }
        }

        // --- ARGOCD ---
        stage('Apply ArgoCD Applications') {
    when { expression { params.ACTION == 'FULL_PIPELINE' } }
    steps {
        script {
            if (fileExists('argocd')) {
                sh """
                    for f in argocd/*.yaml; do
                        [ -f "\$f" ] && sed 's/\\\${ENV}/${ENV}/g' \$f | kubectl apply -f -
                    done
                """
            } else {
                echo "argocd folder not found, skipping ArgoCD deployment."
            }
        }
    }
}


        stage('Sync ArgoCD Apps') {
            when { expression { params.ACTION == 'FULL_PIPELINE' } }
            steps {
                sh """
                    argocd app sync backend --grpc-web
                    argocd app sync database --grpc-web
                    argocd app sync frontend --grpc-web
                """
            }
        }
    }
}
