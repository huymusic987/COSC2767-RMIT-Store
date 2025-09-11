pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        PATH = "${tool 'NodeJS'}/bin:${env.PATH}"
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Debug Info') {
            steps {
                echo "=== JENKINS DEBUG INFORMATION ==="
                echo "Jenkins can read the Jenkinsfile successfully!"
                echo "Build Number: ${env.BUILD_NUMBER}"
                echo "Workspace: ${env.WORKSPACE}"
                echo "Job Name: ${env.JOB_NAME}"
                echo "Build URL: ${env.BUILD_URL}"
                echo "Node Name: ${env.NODE_NAME}"
                
                script {
                    sh '''
                    echo "=== SYSTEM INFORMATION ==="
                    echo "Current user: $(whoami)"
                    echo "Current directory: $(pwd)"
                    echo "Git version: $(git --version)"
                    echo "Docker version: $(docker --version)"
                    echo "Node version: $(node --version || echo 'Node not found')"
                    echo "NPM version: $(npm --version || echo 'NPM not found')"
                    
                    echo "=== REPOSITORY CONTENTS ==="
                    echo "Repository root contents:"
                    ls -la
                    
                    echo "=== GIT INFORMATION ==="
                    echo "Current commit: $(git rev-parse HEAD)"
                    echo "Current branch: $(git branch --show-current)"
                    echo "Recent commits:"
                    git log --oneline -5
                    
                    echo "=== CHECKING FOR CLIENT/SERVER DIRECTORIES ==="
                    if [ -d "./client" ]; then
                        echo "✓ Client directory exists"
                        echo "Client directory contents:"
                        ls -la ./client/ | head -10
                    else
                        echo "✗ Client directory NOT found"
                    fi
                    
                    if [ -d "./server" ]; then
                        echo "✓ Server directory exists"
                        echo "Server directory contents:"
                        ls -la ./server/ | head -10
                    else
                        echo "✗ Server directory NOT found"
                    fi
                    
                    echo "=== DOCKER PERMISSIONS CHECK ==="
                    docker ps || echo "Docker command failed - check permissions"
                    '''
                }
            }
        }

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
                echo 'Checkout completed successfully'
            }
        }

        stage('Pre-build Validation') {
            steps {
                echo 'Validating build requirements...'
                script {
                    sh '''
                    echo "=== PRE-BUILD VALIDATION ==="
                    
                    # Check if Docker is accessible
                    if ! docker info > /dev/null 2>&1; then
                        echo "ERROR: Docker is not accessible"
                        exit 1
                    fi
                    echo "✓ Docker is accessible"
                    
                    # Check if we can authenticate with Docker Hub (optional)
                    echo "Docker Hub login status check..."
                    docker info | grep -i "registry" || echo "No registry info available"
                    
                    # Validate directory structure
                    VALIDATION_FAILED=false
                    
                    if [ ! -d "./client" ]; then
                        echo "WARNING: Client directory not found"
                        # Don't fail build, just warn
                    else
                        if [ ! -f "./client/Dockerfile" ]; then
                            echo "ERROR: Client Dockerfile not found"
                            VALIDATION_FAILED=true
                        else
                            echo "✓ Client Dockerfile found"
                        fi
                    fi
                    
                    if [ ! -d "./server" ]; then
                        echo "WARNING: Server directory not found"
                        # Don't fail build, just warn
                    else
                        if [ ! -f "./server/Dockerfile" ]; then
                            echo "ERROR: Server Dockerfile not found"
                            VALIDATION_FAILED=true
                        else
                            echo "✓ Server Dockerfile found"
                        fi
                    fi
                    
                    if [ "$VALIDATION_FAILED" = true ]; then
                        echo "Pre-build validation failed!"
                        exit 1
                    fi
                    
                    echo "✓ Pre-build validation passed"
                    '''
                }
            }
        }

        stage('Determine Changes') {
            steps {
                echo 'Analyzing repository changes...'
                script {
                    env.BUILD_CLIENT = 'false'
                    env.BUILD_SERVER = 'false'
                    
                    sh '''
                    echo "=== CHANGE DETECTION ==="
                    
                    CURRENT_COMMIT=$(git rev-parse HEAD)
                    LAST_COMMIT=$(git rev-parse HEAD~1 2>/dev/null || echo "")
                    
                    echo "Current commit: $CURRENT_COMMIT"
                    echo "Previous commit: ${LAST_COMMIT:-'None (first build)'}"
                    
                    if [ -z "$LAST_COMMIT" ]; then
                        echo "First run detected - building everything"
                        echo "BUILD_CLIENT=true" > build_flags.env
                        echo "BUILD_SERVER=true" >> build_flags.env
                    else
                        echo "Checking for changes between commits..."
                        
                        # Show all changed files
                        echo "Changed files:"
                        git diff --name-only "$LAST_COMMIT" "$CURRENT_COMMIT" || echo "No changes detected"
                        
                        # Check for client changes
                        CLIENT_CHANGED=$(git diff --name-only "$LAST_COMMIT" "$CURRENT_COMMIT" | grep '^client/' || true)
                        if [ -n "$CLIENT_CHANGED" ]; then
                            echo "✓ Client changes detected:"
                            echo "$CLIENT_CHANGED"
                            echo "BUILD_CLIENT=true" > build_flags.env
                        else
                            echo "○ No client changes detected"
                            echo "BUILD_CLIENT=false" > build_flags.env
                        fi
                        
                        # Check for server changes
                        SERVER_CHANGED=$(git diff --name-only "$LAST_COMMIT" "$CURRENT_COMMIT" | grep '^server/' || true)
                        if [ -n "$SERVER_CHANGED" ]; then
                            echo "✓ Server changes detected:"
                            echo "$SERVER_CHANGED"
                            echo "BUILD_SERVER=true" >> build_flags.env
                        else
                            echo "○ No server changes detected"
                            echo "BUILD_SERVER=false" >> build_flags.env
                        fi
                    fi
                    
                    echo "=== BUILD PLAN ==="
                    cat build_flags.env
                    '''
                    
                    // Read the build flags into environment variables
                    def buildFlags = readFile('build_flags.env').trim()
                    buildFlags.split('\n').each { line ->
                        def (key, value) = line.split('=')
                        env."${key}" = value
                    }
                    
                    echo "Build plan: CLIENT=${env.BUILD_CLIENT}, SERVER=${env.BUILD_SERVER}"
                }
            }
        }

        stage('Build & Push Client') {
            when {
                environment name: 'BUILD_CLIENT', value: 'true'
            }
            steps {
                echo 'Building and pushing client Docker image...'
                script {
                    sh '''
                    echo "=== BUILDING CLIENT IMAGE ==="
                    
                    if [ ! -d "./client" ]; then
                        echo "ERROR: Client directory not found!"
                        exit 1
                    fi
                    
                    CURRENT_COMMIT=$(git rev-parse HEAD)
                    CLIENT_IMAGE="huymusic987/rmit-store-client"
                    
                    echo "Building client image with tag: $CLIENT_IMAGE:$CURRENT_COMMIT"
                    
                    # Build the image
                    docker build -t "$CLIENT_IMAGE:$CURRENT_COMMIT" ./client
                    
                    # Tag as latest
                    docker tag "$CLIENT_IMAGE:$CURRENT_COMMIT" "$CLIENT_IMAGE:latest"
                    
                    echo "✓ Client image built successfully"
                    
                    # Show image info
                    docker images | grep rmit-store-client
                    
                    echo "Pushing client image..."
                    docker push "$CLIENT_IMAGE:$CURRENT_COMMIT"
                    docker push "$CLIENT_IMAGE:latest"
                    
                    echo "✓ Client image pushed successfully"
                    '''
                }
            }
        }

        stage('Build & Push Server') {
            when {
                environment name: 'BUILD_SERVER', value: 'true'
            }
            steps {
                echo 'Building and pushing server Docker image...'
                script {
                    sh '''
                    echo "=== BUILDING SERVER IMAGE ==="
                    
                    if [ ! -d "./server" ]; then
                        echo "ERROR: Server directory not found!"
                        exit 1
                    fi
                    
                    CURRENT_COMMIT=$(git rev-parse HEAD)
                    SERVER_IMAGE="huymusic987/rmit-store-server"
                    
                    echo "Building server image with tag: $SERVER_IMAGE:$CURRENT_COMMIT"
                    
                    # Build the image
                    docker build -t "$SERVER_IMAGE:$CURRENT_COMMIT" ./server
                    
                    # Tag as latest
                    docker tag "$SERVER_IMAGE:$CURRENT_COMMIT" "$SERVER_IMAGE:latest"
                    
                    echo "✓ Server image built successfully"
                    
                    # Show image info
                    docker images | grep rmit-store-server
                    
                    echo "Pushing server image..."
                    docker push "$SERVER_IMAGE:$CURRENT_COMMIT"
                    docker push "$SERVER_IMAGE:latest"
                    
                    echo "✓ Server image pushed successfully"
                    '''
                }
            }
        }

        stage('Cleanup') {
            steps {
                echo 'Cleaning up build artifacts...'
                script {
                    sh '''
                    echo "=== CLEANUP ==="
                    
                    # Remove build flags file
                    rm -f build_flags.env
                    
                    # Clean up old Docker images (keep last 3 versions)
                    echo "Cleaning up old Docker images..."
                    
                    # Clean client images
                    docker images huymusic987/rmit-store-client --format "table {{.Repository}}:{{.Tag}}\t{{.CreatedAt}}" | grep -v latest | tail -n +4 | awk '{print $1}' | xargs -r docker rmi || echo "No old client images to clean"
                    
                    # Clean server images
                    docker images huymusic987/rmit-store-server --format "table {{.Repository}}:{{.Tag}}\t{{.CreatedAt}}" | grep -v latest | tail -n +4 | awk '{print $1}' | xargs -r docker rmi || echo "No old server images to clean"
                    
                    echo "✓ Cleanup completed"
                    '''
                }
            }
        }
    }

    post {
        always {
            echo '=== BUILD SUMMARY ==='
            script {
                sh '''
                echo "Build completed at: $(date)"
                echo "Total build time: ${BUILD_DURATION:-'Unknown'}"
                echo "Final Docker images:"
                docker images | grep rmit-store || echo "No rmit-store images found"
                '''
            }
        }
        success {
            echo '🎉 Pipeline succeeded!'
            echo "Build #${env.BUILD_NUMBER} completed successfully"
        }
        failure {
            echo '❌ Pipeline failed!'
            echo "Build #${env.BUILD_NUMBER} failed - check the logs above for details"
        }
    }
}