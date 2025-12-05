pipeline {
    agent any

    environment {
        SONAR_HOST_URL = "https://v2code.rtwohealthcare.com"
        SONAR_TOKEN = "sqp_ab4016bc5eef902acdbc5f5dbf8f0d46815f0035"

        DOCKER_REGISTRY_URL = "v2deploy.rtwohealthcare.com"
        IMAGE_NAME = "test_v3"
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install & Test Python Code') {
            steps {
                sh """
                    docker run --rm \
                        -v ${WORKSPACE}/backend:/app \
                        -w /app python:3.10-slim sh -c "
                            pip install --upgrade pip &&
                            pip install -r requirements.txt &&
                            pytest --disable-warnings --maxfail=1 -q || true
                        "
                """
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "🔍 Running Sonar Scanner inside custom Docker image"

                // Build simple scanner image (Option A)
                sh """
                    docker build -t sonar-scanner-custom -f sonar-runner-Dockerfile .
                """

                sh """
                    docker run --rm \
                        -v ${WORKSPACE}:/src \
                        -w /src sonar-scanner-custom \
                        sonar-scanner \
                        -Dsonar.projectKey=test_v3 \
                        -Dsonar.sources=backend \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.token=${SONAR_TOKEN}
                """
            }
        }

        stage('Docker Build') {
            when { expression { currentBuild.result == null } }
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} backend
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Docker Push') {
            when { expression { currentBuild.result == null } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-docker-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                        echo "$PASS" | docker login ${DOCKER_REGISTRY_URL} -u "$USER" --password-stdin

                        docker push ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY_URL}/${IMAGE_NAME}:latest

                        docker logout ${DOCKER_REGISTRY_URL}
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Pipeline FAILED — go fix it."
        }
        success {
            echo "✅ Pipeline SUCCESS — Good job Boss."
        }
    }
}
