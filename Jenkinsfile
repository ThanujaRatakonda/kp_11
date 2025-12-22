pipeline {
    agent any
    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_6"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
        ENV = "dev"
    }
    parameters {
        choice(
            name: 'ACTION',
            choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY', 'K8S_ONLY'],
            description: 'Choose FULL_PIPELINE, FRONTEND_ONLY, BACKEND_ONLY, or K8S_ONLY'
        )
        string(name: 'FRONTEND_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for frontend')
        string(name: 'BACKEND_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for backend')
        string(name: 'DB_REPLICA_COUNT', defaultValue: '1', description: 'Replica count for database')
    }

    stages {

        stage('Checkout') {
            when { expression { params.ACTION != 'K8S_ONLY' } }
            steps { git 'https://github.com/ThanujaRatakonda/kp_6.git' }
        }

        stage('Setup Storage') {
            when { expression { params.ACTION == 'FULL_PIPELINE' || params.ACTION == 'K8S_ONLY' } }
            steps {
                echo "Applying StorageClass, PV, PVC (skip if already exists)"
                sh '''
                kubectl get storageclass shared-storage >/dev/null 2>&1 || kubectl apply -f k8s/shared-storage-class.yaml
                kubectl get pv shared-pv >/dev/null 2>&1 || kubectl apply -f k8s/shared-pv.yaml
                kubectl get pvc shared-pvc -n ${ENV} >/dev/null 2>&1 || kubectl apply -f k8s/shared-pvc.yaml -n ${ENV}
                '''
            }
        }

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
            }
        }

        stage('Push Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
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

        stage('Deploy Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY', 'K8S_ONLY'] } }
            steps { sh "kubectl apply -f k8s/frontend-deployment.yaml -n ${ENV}" }
        }

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
            }
        }

        stage('Push Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
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

        stage('Deploy Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY', 'K8S_ONLY'] } }
            steps {
                sh "kubectl apply -f k8s/backend-deployment.yaml -n ${ENV}"
                sh "kubectl apply -f k8s/backend-ingress.yaml -n ${ENV} || true"
            }
        }

        stage('Deploy Database') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'K8S_ONLY'] } }
            steps {
                sh "kubectl apply -f k8s/database-deployment.yaml -n ${ENV} || true"
                sh "kubectl scale statefulset database --replicas=${params.DB_REPLICA_COUNT} -n ${ENV}"
            }
        }

        stage('Scale Frontend & Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'K8S_ONLY', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
            steps {
                sh """
                kubectl scale deployment frontend --replicas=${params.FRONTEND_REPLICA_COUNT} -n ${ENV} || true
                kubectl scale deployment backend --replicas=${params.BACKEND_REPLICA_COUNT} -n ${ENV} || true
                kubectl get deployments -n ${ENV}
                """
            }
        }

        stage('Deploy VPA') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'K8S_ONLY', 'FRONTEND_ONLY', 'BACKEND_ONLY'] } }
            steps {
                sh "kubectl apply -f k8s/frontend-vpa.yaml -n ${ENV} || true"
                sh "kubectl apply -f k8s/backend-vpa.yaml -n ${ENV} || true"
            }
        }
    }
}
