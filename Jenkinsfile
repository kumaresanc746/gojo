pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USERNAME = 'kumaresan05'
        DOCKER_IMAGE_PREFIX = 'grocery-store'
        KUBERNETES_NAMESPACE = 'grocery-store'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Backend Dependencies') {
                    steps {
                        dir('backend') {
                            sh 'npm install'
                        }
                    }
                }
                stage('Frontend Dependencies') {
                    steps {
                        sh 'echo "Frontend is static files – no dependencies"'
                    }
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Backend Image') {
                    steps {
                        sh """
                          docker build -t ${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER} backend
                          docker tag ${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER} ${DOCKER_IMAGE_PREFIX}-backend:latest
                        """
                    }
                }
                stage('Frontend Image') {
                    steps {
                        sh """
                          docker build -t ${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER} frontend
                          docker tag ${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER} ${DOCKER_IMAGE_PREFIX}-frontend:latest
                        """
                    }
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir('backend') {
                    sh 'npm test || true'
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-registry-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                      echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin

                      docker tag ${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER} \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER}
                      docker tag ${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER} \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER}

                      docker push \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER}
                      docker push \$DOCKER_USER/${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                  kubectl create namespace ${KUBERNETES_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                  kubectl apply -f k8s/ -n ${KUBERNETES_NAMESPACE}

                  kubectl set image deployment/backend-deployment backend=${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER} -n ${KUBERNETES_NAMESPACE}
                  kubectl set image deployment/frontend-deployment frontend=${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER} -n ${KUBERNETES_NAMESPACE}

                  kubectl rollout status deployment/backend-deployment -n ${KUBERNETES_NAMESPACE}
                  kubectl rollout status deployment/frontend-deployment -n ${KUBERNETES_NAMESPACE}
                """
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully'
        }
        failure {
            echo '❌ Pipeline failed'
        }
        always {
            sh 'docker image prune -f || true'
        }
    }
}
