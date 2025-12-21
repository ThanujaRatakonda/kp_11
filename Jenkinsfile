pipeline {
    agent any

    parameters {
        choice(
            name: 'ENV',
            choices: ['dev', 'qa', 'prod'],
            description: 'Select target environment'
        )
    }

    environment {
        REPO_URL       = 'https://github.com/ThanujaRatakonda/kp_9'
        IMAGE_TAG      = "${BUILD_NUMBER}"
        HARBOR_URL     = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_9"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: "${REPO_URL}"
            }
        }

        // ---------------- BUILD IMAGES ----------------
        stage('Build Images') {
            steps {
                sh '''
                docker build -t frontend:${IMAGE_TAG} backend/
                docker build -t backend:${IMAGE_TAG} frontend/
                '''
            }
        }

        // ---------------- PUSH TO HARBOR ----------------
        stage('Push Images to Harbor') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'harbor-creds',
                            usernameVariable: 'HARBOR_USER',
                            passwordVariable: 'HARBOR_PASS'
                        )
                    ]) {
                        sh '''
                        echo $HARBOR_PASS | docker login ${HARBOR_URL} \
                          -u $HARBOR_USER --password-stdin

                        docker tag frontend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                        docker tag backend:${IMAGE_TAG}  ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}

                        docker push ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                        docker push ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                        '''
                    }
                }
            }
        }

        // ---------------- UPDATE HELM VALUES ----------------
        stage('Update Helm Image Tags') {
            steps {
                sh '''
                sed -i "s/tag:.*/tag: ${IMAGE_TAG}/" backend-hc/values.yaml
                sed -i "s/tag:.*/tag: ${IMAGE_TAG}/" frontend-hc/values.yaml
                '''
            }
        }

        // ---------------- PREPARE ARGOCD MANIFESTS ----------------
        stage('Prepare Manifests') {
            steps {
                sh '''
                mkdir -p rendered/argocd rendered/k8s

                for f in argocd/*.yaml; do
                  envsubst < $f > rendered/argocd/$(basename $f)
                done

                for f in k8s/*.yaml; do
                  envsubst < $f > rendered/k8s/$(basename $f)
                done
                '''
            }
        }

        // ---------------- DEPLOY INFRA ----------------
        stage('Deploy Infra') {
            steps {
                sh '''
                kubectl apply -f rendered/k8s/
                '''
            }
        }

        // ---------------- DEPLOY APPS (ARGOCD) ----------------
        stage('Deploy Applications via ArgoCD') {
            steps {
                sh '''
                kubectl apply -f rendered/argocd/
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment to ${params.ENV} completed successfully 🚀"
        }
        failure {
            echo "Deployment to ${params.ENV} failed ❌"
        }
    }
}
