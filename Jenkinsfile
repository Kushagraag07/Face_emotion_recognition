pipeline {
    agent any
    
    environment {
        APP_NAME = 'face-tracking-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        CONTAINER_PORT = '80'
        HOST_PORT = '8500'
        TEST_PORT = '8081'
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
                    // Clean up any existing test container
                    sh "docker stop test-container || true"
                    sh "docker rm test-container || true"
                    
                    // Run test container
                    sh "docker run -d --name test-container -p ${TEST_PORT}:${CONTAINER_PORT} ${APP_NAME}:${IMAGE_TAG}"
                    
                    // Increase wait time and add more robust health checking
                    sh "echo 'Waiting for application to start...'"
                    sh '''
                        # Wait up to 30 seconds for the application to respond
                        ATTEMPTS=0
                        MAX_ATTEMPTS=10
                        WAIT_SECONDS=3
                        
                        until curl -s --head http://localhost:${TEST_PORT} | grep -q "HTTP/" || [ $ATTEMPTS -ge $MAX_ATTEMPTS ]
                        do
                            echo "Attempt $((++ATTEMPTS))/$MAX_ATTEMPTS: Waiting for application to start..."
                            sleep $WAIT_SECONDS
                        done
                        
                        if [ $ATTEMPTS -ge $MAX_ATTEMPTS ]; then
                            echo "Application failed to start after $((MAX_ATTEMPTS * WAIT_SECONDS)) seconds"
                            docker logs test-container
                            exit 1
                        fi
                        
                        # Check for actual status code
                        STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${TEST_PORT})
                        echo "Application responded with status code: $STATUS"
                        
                        if [ "$STATUS" -lt 200 ] || [ "$STATUS" -ge 400 ]; then
                            echo "Application is not healthy (Status: $STATUS)"
                            docker logs test-container
                            exit 1
                        fi
                        
                        echo "Application is running and responding with status code $STATUS"
                    '''
                    
                    // Clean up test container
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
                    
                    // Verify deployment was successful
                    sh '''
                        # Wait up to 30 seconds for the application to respond
                        ATTEMPTS=0
                        MAX_ATTEMPTS=10
                        WAIT_SECONDS=3
                        
                        until curl -s --head http://localhost:${HOST_PORT} | grep -q "HTTP/" || [ $ATTEMPTS -ge $MAX_ATTEMPTS ]
                        do
                            echo "Attempt $((++ATTEMPTS))/$MAX_ATTEMPTS: Waiting for application to start..."
                            sleep $WAIT_SECONDS
                        done
                        
                        if [ $ATTEMPTS -ge $MAX_ATTEMPTS ]; then
                            echo "Deployed application failed to start after $((MAX_ATTEMPTS * WAIT_SECONDS)) seconds"
                            docker logs ${APP_NAME}
                            exit 1
                        fi
                        
                        echo "Application is running at http://localhost:${HOST_PORT}"
                    '''
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
            script {
                // Output logs if available to help with debugging
                sh "docker logs ${APP_NAME} || true"
            }
        }
        always {
            // Clean up old Docker images to save space, keeping the currently running one
            sh "docker image prune -f"
            
            // Clean workspace
            cleanWs()
        }
    }
}