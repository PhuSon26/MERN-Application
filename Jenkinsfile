pipeline {
    agent {
        label 'Jenkins-Agent'
    }

    tools {
        jdk 'jdk21'
        nodejs 'nodejs-22'
    }

    environment {
        APP_NAME = "mern-application"
        RELEASE = "1.0.0"
        DOCKER_USER = "phuson0206"
        DOCKER_PASS = 'dockerhub-tokens'
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
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
                  java -version
                  echo "==== Node.js ===="
                  node --version
                  npm --version
                  echo "==== Docker ===="
                  docker version
                  docker compose version
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                  set -e
                  echo "==== Root dependencies ===="
                  npm ci
                  echo "==== Backend dependencies ===="
                  npm ci --prefix backend
                  echo "==== Frontend dependencies ===="
                  npm ci --prefix frontend
                '''
            }
        }

        stage('Checl Backend') {
            steps {
                dir('backend') {
                    sh '''
                      set -e
                      test -f server.js
                      node --check server.js
                    '''
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh '''
                      set -e
                      npm run build
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_USER}/mern-application-backend:${IMAGE_TAG}", "-f backend/Dockerfile backend")
                    docker.build("${DOCKER_USER}/mern-application-frontend:${IMAGE_TAG}", "-f frontend/Dockerfile frontend")
                }
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