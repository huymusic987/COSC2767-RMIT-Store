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
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push Images') {
            steps {
                sh '''#!/bin/bash
                set -euo pipefail

                CURRENT_COMMIT=$(git rev-parse HEAD)
                LAST_COMMIT=$(git rev-parse HEAD~1 2>/dev/null || echo "")

                BUILD_CLIENT=false
                BUILD_SERVER=false

                if [ -z "$LAST_COMMIT" ]; then
                  echo "First run, building everything..."
                  BUILD_CLIENT=true
                  BUILD_SERVER=true
                else
                  CLIENT_CHANGED=$(git diff --name-only "$LAST_COMMIT" "$CURRENT_COMMIT" | grep '^client/' || true)
                  SERVER_CHANGED=$(git diff --name-only "$LAST_COMMIT" "$CURRENT_COMMIT" | grep '^server/' || true)

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
                fi

                if [ "$BUILD_CLIENT" = true ]; then
                  docker build -t huymusic987/rmit-store-client:"$CURRENT_COMMIT" ./client
                  docker tag huymusic987/rmit-store-client:"$CURRENT_COMMIT" huymusic987/rmit-store-client:latest
                  docker push huymusic987/rmit-store-client:"$CURRENT_COMMIT"
                  docker push huymusic987/rmit-store-client:latest
                fi

                if [ "$BUILD_SERVER" = true ]; then
                  docker build -t huymusic987/rmit-store-server:"$CURRENT_COMMIT" ./server
                  docker tag huymusic987/rmit-store-server:"$CURRENT_COMMIT" huymusic987/rmit-store-server:latest
                  docker push huymusic987/rmit-store-server:"$CURRENT_COMMIT"
                  docker push huymusic987/rmit-store-server:latest
                fi
                '''
            }
        }
    }
}
