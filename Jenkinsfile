pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'  // Change to your registry (docker.io, harbor, etc.)
        DOCKER_USERNAME = credentials('kumaresan05')  // Docker registry username
        DOCKER_IMAGE_PREFIX = 'grocery-store'
        KUBERNETES_NAMESPACE = 'grocery-store'
    }
    
    stages {
        stage('Git Pull') {
            steps {
                script {
                    echo 'Pulling latest code from Git repository'
                    sh '''
                        git pull origin main || true
                        git log -1 --oneline
                    '''
                }
            }
        }
        
        stage('Install Dependencies') {
            parallel {
                stage('Backend Dependencies') {
                    steps {
                        script {
                            echo 'Installing backend dependencies'
                            dir('backend') {
                                sh 'npm install'
                            }
                        }
                    }
                }
                stage('Frontend Dependencies') {
                    steps {
                        script {
                            echo 'Checking frontend dependencies'
                            // Frontend is static HTML/CSS/JS, no npm install needed
                            sh 'echo "Frontend is static files, no dependencies to install"'
                        }
                    }
                }
            }
        }
        
        stage('Build Frontend') {
            steps {
                script {
                    echo 'Building frontend static files'
                    dir('frontend') {
                        sh '''
                            echo "Frontend build complete"
                            # In production, you might want to minify CSS/JS here
                        '''
                    }
                }
            }
        }
        
        stage('Docker Build') {
            parallel {
                stage('Build Backend Image') {
                    steps {
                        script {
                            echo 'Building backend Docker image'
                            sh """
                                docker build -t ${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER} ./backend
                                docker tag ${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER} ${DOCKER_IMAGE_PREFIX}-backend:latest
                            """
                        }
                    }
                }
                stage('Build Frontend Image') {
                    steps {
                        script {
                            echo 'Building frontend Docker image'
                            sh """
                                docker build -t ${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER} ./frontend
                                docker tag ${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER} ${DOCKER_IMAGE_PREFIX}-frontend:latest
                            """
                        }
                    }
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                script {
                    echo 'Running backend tests'
                    dir('backend') {
                        sh 'npm test'
                    }
                }
            }
        }
        
        stage('Docker Push') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                }
            }
            steps {
                script {
                    echo 'Pushing Docker images to registry'
                    withCredentials([usernamePassword(credentialsId: 'docker-registry-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin ${DOCKER_REGISTRY}
                            docker tag ${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER} ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER}
                            docker tag ${DOCKER_IMAGE_PREFIX}-backend:latest ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-backend:latest
                            docker tag ${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER} ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER}
                            docker tag ${DOCKER_IMAGE_PREFIX}-frontend:latest ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-frontend:latest
                            docker push ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER}
                            docker push ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-backend:latest
                            docker push ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER}
                            docker push ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-frontend:latest
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                }
            }
            steps {
                script {
                    echo 'Deploying to Kubernetes cluster'
                    sh """
                        # Verify kubectl is configured (assumes kubeconfig is already set up)
                        kubectl cluster-info
                        
                        # Create namespace if not exists
                        kubectl create namespace ${KUBERNETES_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Apply all Kubernetes manifests
                        kubectl apply -f k8s/namespace.yaml
                        kubectl apply -f k8s/mongo-deployment.yaml -n ${KUBERNETES_NAMESPACE}
                        kubectl apply -f k8s/backend-deployment.yaml -n ${KUBERNETES_NAMESPACE}
                        kubectl apply -f k8s/frontend-deployment.yaml -n ${KUBERNETES_NAMESPACE}
                        kubectl apply -f k8s/mongo-express-deployment.yaml -n ${KUBERNETES_NAMESPACE}
                        kubectl apply -f k8s/prometheus-deployment.yaml -n ${KUBERNETES_NAMESPACE}
                        kubectl apply -f k8s/grafana-deployment.yaml -n ${KUBERNETES_NAMESPACE}
                        
                        # Update images to use latest build
                        kubectl set image deployment/backend-deployment backend=${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-backend:${BUILD_NUMBER} -n ${KUBERNETES_NAMESPACE}
                        kubectl set image deployment/frontend-deployment frontend=${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE_PREFIX}-frontend:${BUILD_NUMBER} -n ${KUBERNETES_NAMESPACE}
                        
                        # Wait for rollout
                        kubectl rollout status deployment/backend-deployment -n ${KUBERNETES_NAMESPACE} --timeout=5m
                        kubectl rollout status deployment/frontend-deployment -n ${KUBERNETES_NAMESPACE} --timeout=5m
                        kubectl rollout status deployment/mongo-deployment -n ${KUBERNETES_NAMESPACE} --timeout=5m
                        
                        echo "Deployment completed successfully"
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo 'Performing health checks'
                    sh '''
                        # Wait a bit for services to be ready
                        sleep 10
                        
                        # Check backend health
                        curl -f http://localhost:3000/health || exit 1
                        
                        # Check frontend health
                        curl -f http://localhost/health || exit 1
                        
                        echo "Health checks passed"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
            // Add notifications here (email, Slack, etc.)
        }
        failure {
            echo 'Pipeline failed!'
            // Add failure notifications here
        }
        always {
            script {
                // Cleanup
                sh '''
                    docker system prune -f || true
                '''
            }
        }
    }
}


