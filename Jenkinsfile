pipeline {
    agent any
    
    environment {
        APP_NAME = 'face-tracking-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        CONTAINER_PORT = '8500'
        HOST_PORT = '8500'
        AWS_INSTANCE_IP = sh(script: 'curl -s http://169.254.169.254/latest/meta-data/public-ipv4', returnStdout: true).trim()
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
                    sh "docker tag ${APP_NAME}:${IMAGE_TAG} ${APP_NAME}:latest"
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                script {
                    // Clean up any existing test container
                    sh "docker stop test-${APP_NAME} || true"
                    sh "docker rm test-${APP_NAME} || true"
                    
                    // Run test container with application port mapping to a test port
                    def testPort = "8501"
                    sh "docker run -d --name test-${APP_NAME} -p ${testPort}:${CONTAINER_PORT} ${APP_NAME}:${IMAGE_TAG}"
                    
                    // Wait for container to start and application to be ready
                    sh """
                        # Wait up to 30 seconds for application to be ready
                        ATTEMPTS=0
                        MAX_ATTEMPTS=10
                        WAIT_SECONDS=3
                        
                        echo "Waiting for test container to be ready..."
                        
                        # Ensure container is running
                        until [ "\$(docker inspect -f {{.State.Running}} test-${APP_NAME} 2>/dev/null)" = "true" ] || [ \$ATTEMPTS -ge 3 ]; do
                            ATTEMPTS=\$((ATTEMPTS + 1))
                            echo "Container not running yet, waiting... (\$ATTEMPTS/3)"
                            sleep 2
                        done
                        
                        if [ "\$(docker inspect -f {{.State.Running}} test-${APP_NAME} 2>/dev/null)" != "true" ]; then
                            echo "Test container failed to start"
                            docker logs test-${APP_NAME}
                            exit 1
                        fi
                        
                        # Now check if application is responding
                        ATTEMPTS=0
                        until curl -s --head http://localhost:${testPort} | grep -q "HTTP/" || [ \$ATTEMPTS -ge \$MAX_ATTEMPTS ]
                        do
                            ATTEMPTS=\$((ATTEMPTS + 1))
                            echo "Attempt \$ATTEMPTS/\$MAX_ATTEMPTS: Waiting for application to start..."
                            sleep \$WAIT_SECONDS
                        done
                        
                        if [ \$ATTEMPTS -ge \$MAX_ATTEMPTS ]; then
                            echo "Test application failed to start after \$((MAX_ATTEMPTS * WAIT_SECONDS)) seconds"
                            docker logs test-${APP_NAME}
                            exit 1
                        fi
                        
                        # Get status code
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${testPort})
                        echo "Application responded with status code: \$STATUS"
                        
                        if [ "\$STATUS" -lt 200 ] || [ "\$STATUS" -ge 400 ]; then
                            echo "Application is not healthy (Status: \$STATUS)"
                            docker logs test-${APP_NAME}
                            exit 1
                        fi
                        
                        echo "Test passed: Application is running correctly"
                    """
                    
                    // Clean up test container
                    sh "docker stop test-${APP_NAME} && docker rm test-${APP_NAME}"
                }
            }
        }
        
        stage('Deploy Container') {
            steps {
                script {
                    // Stop and remove any existing container with the same name
                    sh "docker stop ${APP_NAME} || true"
                    sh "docker rm ${APP_NAME} || true"
                    
                    // Run the new container with restart policy for automatic recovery
                    sh "docker run -d --name ${APP_NAME} --restart unless-stopped -p ${HOST_PORT}:${CONTAINER_PORT} ${APP_NAME}:${IMAGE_TAG}"
                    
                    // Verify deployment was successful
                    sh """
                        # Wait up to 30 seconds for the application to respond
                        ATTEMPTS=0
                        MAX_ATTEMPTS=10
                        WAIT_SECONDS=3
                        
                        until curl -s --head http://localhost:${HOST_PORT} | grep -q "HTTP/" || [ \$ATTEMPTS -ge \$MAX_ATTEMPTS ]
                        do
                            ATTEMPTS=\$((ATTEMPTS + 1))
                            echo "Attempt \$ATTEMPTS/\$MAX_ATTEMPTS: Waiting for application to start..."
                            sleep \$WAIT_SECONDS
                        done
                        
                        if [ \$ATTEMPTS -ge \$MAX_ATTEMPTS ]; then
                            echo "Deployed application failed to start after \$((MAX_ATTEMPTS * WAIT_SECONDS)) seconds"
                            docker logs ${APP_NAME}
                            exit 1
                        fi
                        
                        echo "Application is running at http://\${AWS_INSTANCE_IP}:${HOST_PORT}"
                    """
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    // Remove dangling images and unused containers to keep the EC2 instance clean
                    sh "docker system prune -f"
                    
                    // Keep only the last 3 builds to save disk space
                    sh """
                        # Get image IDs for all but the latest 3 builds
                        OLD_IMAGES=\$(docker images ${APP_NAME} --format "{{.ID}} {{.Tag}}" | grep -v 'latest' | sort -k2 -r | tail -n +4 | awk '{print \$1}')
                        
                        # Remove old images if they exist
                        if [ ! -z "\$OLD_IMAGES" ]; then
                            echo "Removing old images: \$OLD_IMAGES"
                            echo \$OLD_IMAGES | xargs docker rmi -f || true
                        else
                            echo "No old images to remove"
                        fi
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo """
            ======================================================
            Deployment completed successfully!
            
            Your Face Tracking application is now running at:
            http://${env.AWS_INSTANCE_IP}:${HOST_PORT}
            
            Build number: ${env.BUILD_NUMBER}
            ======================================================
            """
        }
        failure {
            echo "Deployment failed!"
            script {
                // Output logs if available to help with debugging
                sh "docker logs ${APP_NAME} || true"
            }
        }
        always {
            // Clean workspace
            cleanWs()
        }
    }
}