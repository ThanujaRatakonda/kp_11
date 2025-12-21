pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_9"
    }

    parameters {
        choice(
            name: 'SERVICE',
            choices: ['all', 'frontend', 'backend'],
            description: 'Which service to build'
        )
        string(
            name: 'ENV',
            defaultValue: 'dev',
            description: 'Kubernetes Namespace'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ThanujaRatakonda/kp_9.git'
            }
        }

        /* ================= BACKEND ================= */

        stage('Build & Push Backend') {
            when { expression { params.SERVICE in ['all', 'backend'] } }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-creds',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )
                ]) {
                    sh """
                    docker build -t backend:${IMAGE_TAG} backend
                    docker tag backend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}

                    echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin
                    docker push ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                    docker rmi backend:${IMAGE_TAG} || true
                    """
                }
            }
        }

        /* ================= FRONTEND ================= */

        stage('Build & Push Frontend') {
            when { expression { params.SERVICE in ['all', 'frontend'] } }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-creds',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )
                ]) {
                    sh """
                    docker build -t frontend:${IMAGE_TAG} frontend
                    docker tag frontend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}

                    echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin
                    docker push ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                    docker rmi frontend:${IMAGE_TAG} || true
                    """
                }
            }
        }

        /* ================= HELM VALUES ================= */

        stage('Update Helm values') {
            steps {
                script {
                    if (params.SERVICE in ['all', 'backend']) {
                        sh "sed -i 's/tag:.*/tag: \"${IMAGE_TAG}\"/' backend-hc/backendvalues.yaml"
                    }
                    if (params.SERVICE in ['all', 'frontend']) {
                        sh "sed -i 's/tag:.*/tag: \"${IMAGE_TAG}\"/' frontend-hc/frontendvalues.yaml"
                    }
                }
            }
        }

        /* ================= DEPLOY ================= */

        stage('Deploy K8s & ArgoCD') {
            steps {
                sh """
                kubectl create namespace ${params.ENV} --dry-run=client -o yaml | kubectl apply -f -

                # Apply k8s manifests
                for f in k8s/*; do
                    envsubst < \$f | kubectl apply -n ${params.ENV} -f -
                done

                # Apply ArgoCD apps (they stay in argocd namespace)
                kubectl apply -f argocd/
                """
            }
        }

        /* ================= GIT COMMIT ================= */

        stage('Commit & Push to Git') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'GitHub',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_PASS'
                    )
                ]) {
                    sh """
                    git config user.name "Thanuja"
                    git config user.email "ratakondathanuja@gmail.com"

                    git add .
                    git commit -m "Update image tag to ${IMAGE_TAG}" || echo "Nothing to commit"

                    git push https://\$GIT_USER:\$GIT_PASS@github.com/ThanujaRatakonda/kp_9.git
                    """
                }
            }
        }
    }
}
