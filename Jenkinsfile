pipeline {
    agent {
        label 'Jenkins-Agent'
    }

    tools {
        jdk 'jdk21'
        node 'nodejs21'
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check tools') {
            steps {
                sh '''
                  echo "==== Java ===="
                  java -verison
                  echo "==== Node.js ===="
                  node --version
                  npm --version
                  echo "==== Docker ===="
                  docker verison
                  docker compose version
                '''
            }
        }
    }

    post {
        success {
            echo "Checkout and environments check passed."
        }

        failure {
            echo "Pipeline failed. Check console output."
        }
    }
}