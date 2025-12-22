pipeline {
    agent any

    environment {
        ENV = "dev"                               // 👈 added
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_9"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
    }

    parameters {
        choice(
            name: 'ACTION',
            choices: ['FULL_PIPELINE', 'SCALE_ONLY', 'FRONTEND_ONLY', 'BACKEND_ONLY'],
            description: 'Choose FULL_PIPELINE, SCALE_ONLY, FRONTEND_ONLY, or BACKEND_ONLY'
        )
        string(name: 'FRONTEND_REPLICA_COUNT', defaultValue: '1')
        string(name: 'BACKEND_REPLICA_COUNT', defaultValue: '1')
        string(name: 'DB_REPLICA_COUNT', defaultValue: '1')
    }

    stages {

        stage('Checkout') {
            when { expression { params.ACTION != 'SCALE_ONLY' } }
            steps {
                git 'https://github.com/ThanujaRatakonda/kp_9.git'
            }
        }

        /* =========================
           🔥 BOOTSTRAP (NEW STAGE)
           ========================= */
        stage('Bootstrap K8s & ArgoCD') {
            when { expression { params.ACTION == 'FULL_PIPELINE' } }
            steps {
                sh """
                  echo "Create namespace if not exists"
                  kubectl get ns ${ENV} || kubectl create ns ${ENV}

                  echo "Apply k8s infra (safe re-run)"
                  kubectl apply -f k8s/ -n ${ENV} || true

                  echo "Apply ArgoCD applications (safe re-run)"
                  kubectl apply -f argocd/ -n argocd || true
                """
            }
        }

        /* =========================
           FRONTEND (UNCHANGED)
           ========================= */
        stage('Build Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE','FRONTEND_ONLY'] } }
            steps {
                sh "docker build -t frontend:${IMAGE_TAG} ./frontend"
            }
        }

        stage('Scan Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE','FRONTEND_ONLY'] } }
            steps {
                sh """
                  trivy image frontend:${IMAGE_TAG} \
                  --severity CRITICAL,HIGH \
                  --ignore-unfixed \
                  --scanners vuln \
                  --format json \
                  -o ${TRIVY_OUTPUT_JSON}
                """
            }
        }

        stage('Push Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE','FRONTEND_ONLY'] } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-creds',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh """
                      docker login ${HARBOR_URL} -u $HARBOR_USER -p $HARBOR_PASS
                      docker tag frontend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                      docker push ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                    """
                }
            }
        }

        /* =========================
           BACKEND (UNCHANGED)
           ========================= */
        stage('Build Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE','BACKEND_ONLY'] } }
            steps {
                sh "docker build -t backend:${IMAGE_TAG} ./backend"
            }
        }

        stage('Scan Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE','BACKEND_ONLY'] } }
            steps {
                sh """
                  trivy image backend:${IMAGE_TAG} \
                  --severity CRITICAL,HIGH \
                  --ignore-unfixed \
                  --scanners vuln \
                  --format json \
                  -o ${TRIVY_OUTPUT_JSON}
                """
            }
        }

        stage('Push Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE','BACKEND_ONLY'] } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-creds',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh """
                      docker login ${HARBOR_URL} -u $HARBOR_USER -p $HARBOR_PASS
                      docker tag backend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                      docker push ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                    """
                }
            }
        }

        /* =========================
           SCALING / HPA (UNCHANGED)
           ========================= */
        stage('Scale Frontend & Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE','SCALE_ONLY','FRONTEND_ONLY','BACKEND_ONLY'] } }
            steps {
                sh "kubectl scale deployment frontend --replicas=${params.FRONTEND_REPLICA_COUNT} || true"
                sh "kubectl scale deployment backend --replicas=${params.BACKEND_REPLICA_COUNT} || true"
            }
        }

    }
}
