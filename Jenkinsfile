
pipeline {
    agent { label 'dev' }

    environment {
        IMAGE_NAME = 'two-tier-flask-app'
        COMPOSE_FILE = 'docker-compose.yml'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Code Checkout') {
            steps {
                checkout scm
                sh 'ls -la'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --no-progress .
                '''
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t "$IMAGE_NAME:$BUILD_NUMBER" .'
            }
        }

        stage('Test') {
            steps {
                sh '''
                    python3 -m venv .venv
                    .venv/bin/pip install -r requirements.txt pytest
                    .venv/bin/python -m pytest
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerHubCreds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKERHUB_PASS" |
                          docker login -u "$DOCKERHUB_USER" \
                          --password-stdin

                        docker tag "$IMAGE_NAME:$BUILD_NUMBER" \
                          "$DOCKERHUB_USER/$IMAGE_NAME:$BUILD_NUMBER"

                        docker push \
                          "$DOCKERHUB_USER/$IMAGE_NAME:$BUILD_NUMBER"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker compose -f "$COMPOSE_FILE" \
                      up -d --build flask-app

                    docker compose -f "$COMPOSE_FILE" ps
                '''
            }
        }
    }

    post {
        success {
            echo 'Build, tests, image push and deployment succeeded.'
        }

        failure {
            echo 'Pipeline failed. Review the stage logs.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
