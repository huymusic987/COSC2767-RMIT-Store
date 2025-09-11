pipeline {
    agent any

    environment {
        PATH = "${tool 'NodeJS'}/bin:${env.PATH}"
    }

    options {
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/huymusic987/COSC2767-RMIT-Store.git'
            }
        }

        stage('Build & Push Images') {
            steps {
                sh '''#!/bin/bash
                set -e

                docker login -u huymusic987 -p happyyear27

                # Get the last and current commit SHAs
                LAST_COMMIT=$(git rev-parse HEAD^) || { echo "Failed to get last commit SHA"; exit 1; }
                CURRENT_COMMIT=$(git rev-parse HEAD) || { echo "Failed to get current commit SHA"; exit 1; }

                # Check for changes in client/ and server/ directories
                CLIENT_CHANGED=$(git diff --name-only "$LAST_COMMIT" "$CURRENT_COMMIT" | grep '^client/' || true)
                SERVER_CHANGED=$(git diff --name-only "$LAST_COMMIT" "$CURRENT_COMMIT" | grep '^server/' || true)

                BUILD_CLIENT=false
                BUILD_SERVER=false

                if [ -n "$CLIENT_CHANGED" ]; then
                  echo "Changes detected in client/. Building client..."
                  BUILD_CLIENT=true
                else
                  echo "No client changes."
                fi

                if [ -n "$SERVER_CHANGED" ]; then
                  echo "Changes detected in server/. Building server..."
                  BUILD_SERVER=true
                else
                  echo "No server changes."
                fi

                if [ "$BUILD_CLIENT" = true ]; then
                  docker rmi huymusic987/rmit-store-client:"$LAST_COMMIT" 2>/dev/null || true
                  docker rmi huymusic987/rmit-store-client:latest 2>/dev/null || true

                  cd client
                  docker build -t huymusic987/rmit-store-client:"$CURRENT_COMMIT" .
                  docker tag huymusic987/rmit-store-client:"$CURRENT_COMMIT" huymusic987/rmit-store-client:latest
                  docker push huymusic987/rmit-store-client:"$CURRENT_COMMIT"
                  cd ..
                fi

                if [ "$BUILD_SERVER" = true ]; then
                  docker rmi huymusic987/rmit-store-server:"$LAST_COMMIT" 2>/dev/null || true
                  docker rmi huymusic987/rmit-store-server:latest 2>/dev/null || true

                  cd server
                  docker build -t huymusic987/rmit-store-server:"$CURRENT_COMMIT" .
                  docker tag huymusic987/rmit-store-server:"$CURRENT_COMMIT" huymusic987/rmit-store-server:latest
                  docker push huymusic987/rmit-store-server:"$CURRENT_COMMIT"
                  cd ..
                fi

                '''
            }
        }
    }
}
