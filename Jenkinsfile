pipeline {
    agent any

    environment {
        APP_NAME = 'rosterup'
        IMAGE_NAME = 'rosterup-devops'
        IMAGE_TAG = "${BUILD_NUMBER}"
        STAGING_CONTAINER = 'rosterup-staging'
        PRODUCTION_CONTAINER = 'rosterup-production'
    }

    stages {

        stage('Build') {
            steps {
                echo '===== BUILD STAGE ====='

                sh 'node --version'
                sh 'npm --version'

                echo 'Installing dependencies...'
                sh 'npm ci'

                echo 'Building Docker image...'
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'

                echo 'Build completed successfully.'
            }
        }

        stage('Test') {
            steps {
                echo '===== TEST STAGE ====='

                sh 'npm test'

                echo 'All automated tests completed successfully.'
            }
        }

        stage('Code Quality') {
            steps {
                echo '===== CODE QUALITY STAGE ====='

                echo 'Running SonarQube analysis...'

                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=rosterup-devops \
                          -Dsonar.projectName=RosterUp-DevOps \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=node_modules/**,coverage/**,test/**
                    '''
                }
            }
        }

        stage('Security') {
            steps {
                echo '===== SECURITY STAGE ====='

                echo 'Running npm security audit...'

                sh '''
                    npm audit --json > npm-audit-report.json || true
                '''

                archiveArtifacts artifacts: 'npm-audit-report.json',
                                 allowEmptyArchive: true

                echo 'Security audit completed.'
            }
        }

        stage('Deploy') {
            steps {
                echo '===== DEPLOY STAGE ====='

                echo 'Removing previous staging container if it exists...'
                sh '''
                    docker rm -f ${STAGING_CONTAINER} || true
                '''

                echo 'Deploying RosterUp to staging...'
                sh '''
                    docker run -d \
                      --name ${STAGING_CONTAINER} \
                      -p 3001:3000 \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                '''

                echo 'Waiting for staging application...'
                sleep 10

                echo 'Checking staging container...'
                sh 'docker ps --filter "name=${STAGING_CONTAINER}"'
            }
        }

        stage('Release') {
            steps {
                echo '===== RELEASE STAGE ====='

                echo 'Removing previous production container if it exists...'
                sh '''
                    docker rm -f ${PRODUCTION_CONTAINER} || true
                '''

                echo 'Releasing RosterUp to production...'
                sh '''
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:production
                '''

                sh '''
                    docker run -d \
                      --name ${PRODUCTION_CONTAINER} \
                      -p 3000:3000 \
                      ${IMAGE_NAME}:production
                '''

                echo 'Production release completed.'
            }
        }

        stage('Monitoring') {
            steps {
                echo '===== MONITORING STAGE ====='

                echo 'Checking production container status...'
                sh 'docker ps --filter "name=${PRODUCTION_CONTAINER}"'

                echo 'Checking application health...'
                sh '''
                    curl -f http://localhost:3000 || \
                    (echo "Production health check failed" && exit 1)
                '''

                echo 'Production monitoring/health check passed.'
            }
        }
    }

    post {
        success {
            echo '========================================='
            echo 'ROSTERUP DEVOPS PIPELINE SUCCESSFUL'
            echo 'All 7 stages completed.'
            echo '========================================='
        }

        failure {
            echo '========================================='
            echo 'ROSTERUP DEVOPS PIPELINE FAILED'
            echo 'Check the failed stage in Jenkins.'
            echo '========================================='
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}