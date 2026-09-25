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
            withEnv(["PATH+SONAR=${tool 'SonarScanner'}/bin"]) {
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

        echo 'Removing previous staging container...'
        sh '''
            docker rm -f rosterup-staging || true
        '''

        echo 'Deploying RosterUp to staging...'
        withCredentials([string(credentialsId: 'rosterup-jwt-secret', variable: 'JWT_SECRET_VALUE')]) {
            sh '''
                docker run -d \
                  --name rosterup-staging \
                  -p 3001:3000 \
                  -e MONGO_URI=mongodb://host.docker.internal:27017/RUDatabase \
                  -e JWT_SECRET="$JWT_SECRET_VALUE" \
                  -e PORT=3000 \
                  ${IMAGE_NAME}:${IMAGE_TAG}
            '''
        }

        echo 'Waiting for staging application...'
        sleep 15

        echo 'Checking staging container...'
        sh '''
            docker ps --filter "name=rosterup-staging" --format "{{.Names}} {{.Status}}"
        '''

        echo 'Checking staging application health...'
        sh '''
            docker inspect -f "{{.State.Running}}" rosterup-staging | grep true
        '''
    }
}

stage('Release') {
    steps {
        echo '===== RELEASE STAGE ====='

        echo 'Removing previous production container...'
        sh '''
            docker rm -f rosterup-production || true
        '''

        echo 'Creating production image tag...'
        sh '''
            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:production
        '''

        echo 'Releasing RosterUp to production...'
        withCredentials([string(credentialsId: 'rosterup-jwt-secret', variable: 'JWT_SECRET_VALUE')]) {
            sh '''
                docker run -d \
                  --name rosterup-production \
                  -p 3002:3000 \
                  -e MONGO_URI=mongodb://host.docker.internal:27017/RUDatabase \
                  -e JWT_SECRET="$JWT_SECRET_VALUE" \
                  -e PORT=3000 \
                  ${IMAGE_NAME}:production
            '''
        }

        echo 'Waiting for production application...'
        sleep 15

        echo 'Checking production container...'
        sh '''
            docker ps --filter "name=rosterup-production" --format "{{.Names}} {{.Status}}"
        '''

        echo 'Production release verified.'
    }
}

stage('Monitoring') {
    steps {
        echo '===== MONITORING STAGE ====='

        echo 'Checking production container status...'
        sh '''
            docker inspect -f "{{.State.Running}}" rosterup-production | grep true
        '''

        echo 'Checking production application...'
        sh '''
            curl -f http://host.docker.internal:3002 || \
            (echo "Production health check failed" && exit 1)
        '''

        echo 'Production monitoring and health check passed.'
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