@Library('Shared') _
pipeline {
    agent {
        label 'Jenkins-Agent'
    }

    tools {
        jdk 'jdk21'
        nodejs 'nodejs-22'
    }

    environment {
        APP_NAME = 'mern-application'
        RELEASE = '1.0.0'
        BACKEND_IMAGE = 'phuson0206/mern-application-backend'
        FRONTEND_IMAGE = 'phuson0206/mern-application-frontend'
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {
        stage('Workspace Cleanup') {
            steps {
                cleanWs()
            }
        }

        stage('Git: Code Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    set -e
                    npm ci --prefix backend
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

        stage('Trivy: Filesystem Scan') {
            steps {
                script {
                    trivy_scan()
                }
            }
        }

        stage('OWASP: Dependency Check') {
            steps {
                dependencyCheck(
                    odcInstallation: 'OWASP',
                    nvdCredentialsId: 'nvd-api-key',
                    additionalArguments: '''
                        --scan backend
                        --scan frontend
                        --format HTML
                        --format XML
                        --out reports/dependency-check
                    '''
                )

                dependencyCheckPublisher(pattern: 'reports/dependency-check/dependency-check-report.xml')
            }
        }

        stage('SonarQube: Code Analysis') {
            steps {
                script {
                    sonarqube_analysis(
                        'SonarQube',
                        'mern-application',
                        'mern-application'
                    )
                }
            }
        }

        stage('SonarQube: Quality Gate') {
            steps {
                script {
                    sonarqube_code_quality()
                }
            }
        }

        stage('Frontend: Build') {
            steps {
                dir('frontend') {
                    sh 'npm run build'
                }
            }
        }

        stage('Docker: Build Images') {
            steps {
                script {
                    docker.build(
                        "${BACKEND_IMAGE}:${IMAGE_TAG}",
                        '-f backend/Dockerfile backend'
                    )

                    docker.build(
                        "${FRONTEND_IMAGE}:${IMAGE_TAG}",
                        '-f frontend/Dockerfile frontend'
                    )
                }
            }
        }

        stage('Trivy: Image Scan') {
            steps {
                sh '''
                    set -e
                    trivy image --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 --no-progress \
                        "${BACKEND_IMAGE}:${IMAGE_TAG}"

                    trivy image --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 --no-progress \
                        "${FRONTEND_IMAGE}:${IMAGE_TAG}"
                '''
            }
        }

        stage('Docker: Push to DockerHub') {
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-tokens') {
                        docker.image("${BACKEND_IMAGE}:${IMAGE_TAG}").push()

                        docker.image("${BACKEND_IMAGE}:${IMAGE_TAG}").push('latest')

                        docker.image("${FRONTEND_IMAGE}:${IMAGE_TAG}").push()

                        docker.image("${FRONTEND_IMAGE}:${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                sh '''
                    docker rmi "${BACKEND_IMAGE}:${IMAGE_TAG}"
                    docker rmi "${FRONTEND_IMAGE}:${IMAGE_TAG}"
                '''
            }
        }
    }

    post {
        success {
            echo 'MERN CI pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Console Output.'
        }

        always {
            cleanWs(
                cleanWhenFailure: false,
                notFailBuild: true
            )
        }
    }
}