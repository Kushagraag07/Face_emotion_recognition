pipeline {
    agent any
    
    environment {
        APP_NAME = 'face-tracking-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        CONTAINER_PORT = '80'
        HOST_PORT = '8500'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${APP_NAME}:${IMAGE_TAG} ."
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                script {
                    // Simple test - check if the Docker container starts correctly
                    sh "docker run -d --name test-container -p 8500:${CONTAINER_PORT} ${APP_NAME}:${IMAGE_TAG}"
                    sh "sleep 5" // Give the container time to start
                    sh "curl -s --head http://localhost:8500 | grep '200 OK' || exit 1"
                    sh "docker stop test-container && docker rm test-container"
                }
            }
        }
        
        stage('Deploy Container') {
            steps {
                script {
                    // Stop and remove any existing container with the same name
                    sh "docker stop ${APP_NAME} || true"
                    sh "docker rm ${APP_NAME} || true"
                    
                    // Run the new container
                    sh "docker run -d --name ${APP_NAME} -p ${HOST_PORT}:${CONTAINER_PORT} ${APP_NAME}:${IMAGE_TAG}"
                    
                    // Tag as latest for easy reference
                    sh "docker tag ${APP_NAME}:${IMAGE_TAG} ${APP_NAME}:latest"
                }
            }
        }
    }
    
    post {
        success {
            echo "Deployment completed successfully! Application is running at http://localhost:${HOST_PORT}"
        }
        failure {
            echo "Deployment failed!"
        }
        always {
            // Clean up old Docker images to save space, keeping the currently running one
            sh "docker image prune -f"
            
            // Clean workspace
            cleanWs()
        }
    }
}