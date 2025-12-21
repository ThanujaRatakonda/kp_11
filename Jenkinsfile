pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'],
            description: 'Pipeline execution type'
        )
        choice(
            name: 'ENV',
            choices: ['dev', 'qa', 'prod'],
            description: 'Target environment / namespace'
        )
    }

    environment {
        REGISTRY = "10.131.103.92:8090"
        PROJECT  = "kp_9"
        ENV      = "${params.ENV}"
    }

    stages {

        stage('Checkout') {
            when { expression { params.ACTION != 'SCALE_ONLY' } }
            steps {
                git branch: 'master', url: 'https://github.com/ThanujaRatakonda/kp_9'
            }
        }

        stage('Calculate Release Version') {
            when { expression { params.ACTION != 'SCALE_ONLY' } }
            steps {
                script {
                    def lastTag = sh(
                        script: "git tag --sort=-v:refname | head -n 1 || true",
                        returnStdout: true
                    ).trim()

                    if (!lastTag) {
                        env.RELEASE = "v1.0.0"
                    } else {
                        def v = lastTag.replace("v","").tokenize(".")
                        env.RELEASE = "v${v[0]}.${v[1]}.${v[2].toInteger()+1}"
                    }

                    echo "Release: ${env.RELEASE}"
                }
            }
        }

        stage('Build & Push Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'H_USER', passwordVariable: 'H_PASS')]) {
                    sh """
                        echo \$H_PASS | docker login ${REGISTRY} -u \$H_USER --password-stdin
                        docker build -t ${REGISTRY}/${PROJECT}/backend:${RELEASE} backend
                        docker push ${REGISTRY}/${PROJECT}/backend:${RELEASE}
                    """
                }
            }
        }

        stage('Build & Push Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'H_USER', passwordVariable: 'H_PASS')]) {
                    sh """
                        echo \$H_PASS | docker login ${REGISTRY} -u \$H_USER --password-stdin
                        docker build -t ${REGISTRY}/${PROJECT}/frontend:${RELEASE} frontend
                        docker push ${REGISTRY}/${PROJECT}/frontend:${RELEASE}
                    """
                }
            }
        }

        stage('Update Helm Values') {
            when { expression { params.ACTION != 'SCALE_ONLY' } }
            steps {
                sh """
                    sed -i 's/tag:.*/tag: "${RELEASE}"/' backend-hc/backendvalues.yaml
                    sed -i 's/tag:.*/tag: "${RELEASE}"/' frontend-hc/frontendvalues.yaml
                """
            }
        }

        stage('Apply Infra & ArgoCD') {
            when { expression { params.ACTION != 'SCALE_ONLY' } }
            steps {
                sh """
                    export ENV=${ENV}

                    # Apply k8s infra (Namespace, PV, PVC, StorageClass)
                    for f in k8s/*.yaml; do
                        envsubst < \$f | kubectl apply -f -
                    done

                    # Apply ArgoCD applications
                    for f in argocd/*.yaml; do
                        envsubst < \$f | kubectl apply -f -
                    done
                """
            }
        }

        stage('Create Git Tag') {
            when { expression { params.ACTION != 'SCALE_ONLY' } }
            steps {
                sh """
                    git tag ${RELEASE}
                    git push origin ${RELEASE}
                    git push https://${GIT_USER}:${GIT_PASS}@github.com/ThanujaRatakonda/kp_9.git ${RELEASE}
                """
            }
        }
    }
}
