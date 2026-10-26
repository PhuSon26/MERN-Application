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
        BACKEND_IMAGE = "phuson0206/mern-application-backend"
        FRONTEND_IMAGE = "phuson0206/mern-application-frontend"
        DOCKER_USER = 'phuson0206'
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

                    echo "==== Backend dependencies ===="
                    npm ci --prefix backend

                    echo "==== Frontend dependencies ===="
                    npm ci --prefix frontend
                '''
            }
        }

        stage('Check Backend') {
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

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=mern-application \
                              -Dsonar.projectName=mern-application \
                              -Dsonar.sources=backend,frontend/src \
                              -Dsonar.sourceEncoding=UTF-8 \
                              -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/coverage/**,**/.scannerwork/**
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()

                        if (qg.status != 'OK') {
                            error "Quality Gate failed: ${qg.status}"
                        }
                    }
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
                    docker.build("${BACKEND_IMAGE}:${IMAGE_TAG}", "-f backend/Dockerfile backend")
                    docker.build("${FRONTEND_IMAGE}:${IMAGE_TAG}", "-f frontend/Dockerfile frontend")
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                  set -e
                  trivy image \
                    --scanners vuln \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    --no-progress \
                    "${BACKEND_IMAGE}:${IMAGE_TAG}"

                  trivy image \
                    --scanners vuln \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    --no-progress \
                    "${FRONTEND_IMAGE}:${IMAGE_TAG}"                  
                '''
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-tokens') {
                        def backend = docker.image("${BACKEND_IMAGE}:${IMAGE_TAG}")
                        def frontend = docker.image("${FRONTEND_IMAGE}:${IMAGE_TAG}")
                        backend.push()
                        backend.push('latest')
                        frontend.push()
                        frontend.push('latest')
                    }
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                sh '''
                  docker rmi ${BACKEND_IMAGE}:$IMAGE_TAG}
                  docker rmi ${FRONTEND_IMAGE}:$IMAGE_TAG}
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