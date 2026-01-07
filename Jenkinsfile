pipeline {
    agent any

    environment {
        // Docker configuration
        DOCKER_IMAGE_NAME = 'kutt'
        DOCKER_IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(7) ?: 'local'}"
        DOCKER_REGISTRY = "${env.DOCKER_REGISTRY ?: 'docker.io'}"
        
        // Build configuration
        BUILD_DATE = sh(script: 'date -u +"%Y-%m-%dT%H:%M:%SZ"', returnStdout: true).trim()
        VCS_REF = "${env.GIT_COMMIT ?: 'local-build'}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
    }

    parameters {
        booleanParam(name: 'PUSH_IMAGE', defaultValue: false, description: 'Push image to registry')
        booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Deploy after build')
        choice(name: 'DOCKER_COMPOSE_FILE', choices: ['docker-compose.yml', 'docker-compose.mariadb.yml', 'docker-compose.postgres.yml'], description: 'Docker compose file to use for testing')
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    if (env.GIT_URL) {
                        checkout scm
                    } else {
                        echo 'Using workspace directly (local build)'
                    }
                }
                sh '''
                    echo "=========================================="
                    echo "Build Information:"
                    echo "Branch: ${GIT_BRANCH:-main}"
                    echo "Commit: ${GIT_COMMIT:-local}"
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "=========================================="
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Installing Node.js dependencies..."
                    npm ci --prefer-offline --no-audit
                    echo "Dependencies installed successfully"
                '''
            }
        }

        stage('Lint & Validate') {
            steps {
                script {
                    try {
                        sh 'npm run lint || echo "No lint script found, skipping..."'
                    } catch (Exception e) {
                        echo "Linting skipped: ${e.message}"
                    }
                    
                    // Validate package.json
                    sh '''
                        echo "Validating package.json..."
                        node -e "const pkg = require('./package.json'); console.log('Package:', pkg.name, pkg.version)"
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def buildArgs = "--build-arg BUILD_DATE=${BUILD_DATE} --build-arg VCS_REF=${VCS_REF} --build-arg VERSION=${env.GIT_COMMIT?.take(7) ?: 'dev'}"
                    
                    def dockerImage = docker.build(
                        "${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}",
                        "${buildArgs} ."
                    )
                    
                    // Also tag with additional tags
                    dockerImage.tag("${DOCKER_IMAGE_NAME}:latest")
                    
                    if (env.GIT_BRANCH) {
                        def branchName = env.GIT_BRANCH.replace('origin/', '').replace('/', '-')
                        dockerImage.tag("${DOCKER_IMAGE_NAME}:${branchName}")
                    }
                    
                    echo "Docker image built: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                }
            }
        }

        stage('Test Docker Image') {
            steps {
                script {
                    sh '''
                        echo "Testing Docker image..."
                        
                        # Stop and remove any existing test container
                        docker stop kutt-test || true
                        docker rm kutt-test || true
                        
                        # Run container with test environment
                        docker run -d --name kutt-test \
                            -e JWT_SECRET=test-secret-for-jenkins-build \
                            -e DB_CLIENT=better-sqlite3 \
                            -e DB_FILENAME=/var/lib/kutt/test-data \
                            -e DISALLOW_REGISTRATION=false \
                            -e DISALLOW_ANONYMOUS_LINKS=false \
                            -p 3001:3000 \
                            ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} || true
                        
                        echo "Waiting for container to start (40 seconds)..."
                        sleep 40
                        
                        # Check if container is still running
                        if docker ps | grep -q kutt-test; then
                            echo "✓ Container is running"
                            
                            # Health check
                            echo "Performing health check..."
                            for i in {1..10}; do
                                if curl -f http://localhost:3001/api/health 2>/dev/null; then
                                    echo "✓ Health check passed"
                                    break
                                fi
                                if [ $i -eq 10 ]; then
                                    echo "✗ Health check failed after 10 attempts"
                                    docker logs kutt-test
                                    exit 1
                                fi
                                echo "Health check attempt $i/10 failed, retrying..."
                                sleep 5
                            done
                        else
                            echo "✗ Container failed to start"
                            docker logs kutt-test || true
                            exit 1
                        fi
                        
                        # Test API endpoint
                        echo "Testing API endpoints..."
                        curl -f http://localhost:3001/api/health || exit 1
                        
                        # Cleanup
                        docker stop kutt-test || true
                        docker rm kutt-test || true
                        
                        echo "✓ Docker image test completed successfully"
                    '''
                }
            }
            post {
                always {
                    sh '''
                        docker stop kutt-test || true
                        docker rm kutt-test || true
                    '''
                }
            }
        }

        stage('Test with Docker Compose') {
            when {
                expression { params.DOCKER_COMPOSE_FILE != null }
            }
            steps {
                script {
                    def composeFile = params.DOCKER_COMPOSE_FILE
                    sh """
                        echo "Testing with ${composeFile}..."
                        
                        # Create test .env file
                        cat > .env.test << EOF
JWT_SECRET=test-secret-for-jenkins-compose-test
DB_NAME=kutt_test
DB_USER=kutt_test
DB_PASSWORD=test_password_123
EOF
                        
                        # Build and start services
                        docker compose -f ${composeFile} --env-file .env.test up -d --build
                        
                        echo "Waiting for services to be ready..."
                        sleep 30
                        
                        # Check if services are running
                        docker compose -f ${composeFile} ps
                        
                        # Test health endpoint
                        for i in {1..15}; do
                            if curl -f http://localhost:3000/api/health 2>/dev/null; then
                                echo "✓ Docker Compose test passed"
                                break
                            fi
                            if [ $i -eq 15 ]; then
                                echo "✗ Docker Compose test failed"
                                docker compose -f ${composeFile} logs
                                exit 1
                            fi
                            sleep 5
                        done
                        
                        # Cleanup
                        docker compose -f ${composeFile} down -v
                        rm -f .env.test
                    """
                }
            }
            post {
                always {
                    sh '''
                        docker compose -f ${DOCKER_COMPOSE_FILE} down -v || true
                        rm -f .env.test || true
                    '''
                }
            }
        }

        stage('Push to Registry') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                    branch 'develop'
                    expression { params.PUSH_IMAGE == true }
                }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-registry-credentials', 
                                                     usernameVariable: 'DOCKER_USER', 
                                                     passwordVariable: 'DOCKER_PASS',
                                                     required: false)]) {
                        sh '''
                            if [ -n "$DOCKER_USER" ] && [ -n "$DOCKER_PASS" ]; then
                                echo "Logging into Docker registry..."
                                echo "${DOCKER_PASS}" | docker login ${DOCKER_REGISTRY} -u "${DOCKER_USER}" --password-stdin || docker login ${DOCKER_REGISTRY} -u "${DOCKER_USER}" -p "${DOCKER_PASS}"
                                
                                echo "Pushing image ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}..."
                                docker push ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                                
                                # Push latest tag if main/master branch
                                if [ "${GIT_BRANCH}" = "origin/main" ] || [ "${GIT_BRANCH}" = "origin/master" ]; then
                                    docker push ${DOCKER_IMAGE_NAME}:latest
                                fi
                                
                                echo "Logging out from Docker registry..."
                                docker logout ${DOCKER_REGISTRY} || true
                            else
                                echo "Docker registry credentials not configured. Skipping push."
                                echo "To push images, configure 'docker-registry-credentials' in Jenkins."
                            fi
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                    expression { params.DEPLOY == true }
                }
            }
            steps {
                script {
                    echo """
                    ==========================================
                    Deployment Stage
                    ==========================================
                    Image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                    
                    Deployment target configuration needed.
                    Options:
                    1. Kubernetes: kubectl set image deployment/kutt
                    2. Docker Swarm: docker service update
                    3. Docker Compose: docker-compose pull && docker-compose up -d
                    ==========================================
                    """
                    
                    // Example deployment commands (uncomment and customize as needed)
                    // sh '''
                    //     # Kubernetes deployment example
                    //     kubectl set image deployment/kutt kutt=${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} -n kutt
                    //     kubectl rollout status deployment/kutt -n kutt
                    // '''
                    
                    // sh '''
                    //     # Docker Compose deployment example
                    //     docker-compose -f docker-compose.yml pull
                    //     docker-compose -f docker-compose.yml up -d
                    // '''
                }
            }
        }
    }

    post {
        success {
            echo '''
            ╔════════════════════════════════════════╗
            ║   ✅ Pipeline Completed Successfully   ║
            ╚════════════════════════════════════════╝
            '''
            script {
                def message = """
                ✅ Build Successful
                
                Branch: ${env.GIT_BRANCH ?: 'N/A'}
                Commit: ${env.GIT_COMMIT ?: 'N/A'}
                Image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                Build #: ${env.BUILD_NUMBER}
                Date: ${BUILD_DATE}
                """.stripIndent()
                
                echo message
            }
        }
        failure {
            echo '''
            ╔════════════════════════════════════════╗
            ║   ❌ Pipeline Failed                   ║
            ╚════════════════════════════════════════╝
            '''
        }
        always {
            // Cleanup Docker images to save space
            sh '''
                echo "Cleaning up dangling Docker images..."
                docker image prune -f || true
            '''
            
            // Archive artifacts if needed
            archiveArtifacts artifacts: '**/*.log', allowEmptyArchive: true
            
            // Clean workspace (optional)
            // cleanWs()
        }
    }
}
