
pipeline {
    agent any

    parameters {
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Target environment namespace')
    }

    environment {
        REGISTRY = '10.131.103.92:8090'
        PROJECT  = 'kp_9'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                    sh 'docker login ${REGISTRY} -u $HARBOR_USER -p $HARBOR_PASS'
                }
            }
        }

        stage('Build & Push Backend') {
            steps {
                sh '''
                docker build -t ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} backend/
                docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
                '''
            }
        }

        stage('Build & Push Frontend') {
            steps {
                sh '''
                docker build -t ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} frontend/
                docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
                '''
            }
        }

        stage('Apply K8s Base') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                    export KUBECONFIG=$KUBECONFIG_FILE
                    for f in $(ls k8s/*); do
                      ENV=${ENV} envsubst < $f | kubectl apply -f -
                    done
                    '''
                }
            }
        }

        stage('Update Helm Values & Push') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GHTOKEN')]) {
                    sh '''
                    sed -i "s|tag:.*|tag: \\"${IMAGE_TAG}\\"|" backend-hc/backendvalues.yaml
                    sed -i "s|tag:.*|tag: \\"${IMAGE_TAG}\\"|" frontend-hc/frontendvalues.yaml

                    git config user.email "ci@example.com"
                    git config user.name "ci-bot"
                    git add backend-hc/backendvalues.yaml frontend-hc/frontendvalues.yaml
                    git commit -m "Update image tag to ${IMAGE_TAG} for ${ENV}"
                    git push https://$GHTOKEN@github.com/ThanujaRatakonda/kp_9.git HEAD:master
                    '''
                }
            }
        }

        stage('Apply ArgoCD Apps') {
            steps {
                sh '''
                export KUBECONFIG=$KUBECONFIG_FILE
                for f in $(ls argocd/*); do
                  ENV=${ENV} envsubst < $f | kubectl apply -n argocd -f -
                done
                '''
            }
        }
    }
}
