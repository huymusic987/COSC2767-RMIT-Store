pipeline {
    agent any
    
    stages {
        stage('Test') {
            steps {
                echo 'Hello from Jenkins!'
                echo "Build number: ${env.BUILD_NUMBER}"
                sh 'pwd'
                sh 'ls -la'
            }
        }
    }
}