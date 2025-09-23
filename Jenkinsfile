pipeline {
    agent any

    // กำหนด Environment Variables
    environment {
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'
        N8N_WEBHOOK_URL_CREDENTIALS = 'n8n-webhook-url'
        DOCKER_IMAGE_NAME = 'iamsamitdev/flask-docker-app'
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        // Stage 1: ดึงโค้ดล่าสุดจาก Git
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }

        // Stage 2: ติดตั้ง Dependencies และรัน Tests
        stage('Install Dependencies and Test') {
            steps {
                script {
                    echo 'Installing Python dependencies...'
                    sh '''
                        python3 -m pip install --upgrade pip
                        pip install -r requirements.txt
                    '''
                    
                    echo 'Running tests...'
                    sh '''
                        python3 -m pytest tests/ -v --tb=short --junitxml=test-results.xml
                    '''
                }
            }
            post {
                always {
                    // บันทึกผลการทดสอบ
                    junit 'test-results.xml'
                }
            }
        }

        // Stage 3: สร้าง Docker Image
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                    docker.build("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}", '.')
                }
            }
        }

        // Stage 4: Push Image ไปยัง Docker Hub
        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', DOCKERHUB_CREDENTIALS) {
                        echo "Pushing image to Docker Hub..."
                        docker.image("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}").push()
                        docker.image("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}").push('latest')
                    }
                }
            }
        }
    }

    // Post Actions: ส่วนแจ้งเตือน N8N
    post {
        success {
            script {
                echo 'CI process completed successfully. Notifying N8N...'
                withCredentials([string(credentialsId: N8N_WEBHOOK_URL_CREDENTIALS, variable: 'WEBHOOK_URL')]) {
                    sh """
                        curl -X POST -H "Content-Type: application/json" \\
                        -d '{
                            "status": "SUCCESS",
                            "project": "${JOB_NAME}",
                            "buildNumber": "${BUILD_NUMBER}",
                            "imageUrl": "${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}",
                            "dockerHubUrl": "https://hub.docker.com/r/${DOCKER_IMAGE_NAME}/tags",
                            "testsStatus": "PASSED",
                            "buildUrl": "${BUILD_URL}"
                        }' \\
                        '${WEBHOOK_URL}'
                    """
                }
            }
        }
        failure {
            script {
                echo 'CI process failed. Notifying N8N...'
                withCredentials([string(credentialsId: N8N_WEBHOOK_URL_CREDENTIALS, variable: 'WEBHOOK_URL')]) {
                    sh """
                        curl -X POST -H "Content-Type: application/json" \\
                        -d '{
                            "status": "FAILED",
                            "project": "${JOB_NAME}",
                            "buildNumber": "${BUILD_NUMBER}",
                            "buildUrl": "${BUILD_URL}",
                            "testsStatus": "FAILED"
                        }' \\
                        '${WEBHOOK_URL}'
                    """
                }
            }
        }
        unstable {
            script {
                echo 'Tests failed but build succeeded. Notifying N8N...'
                withCredentials([string(credentialsId: N8N_WEBHOOK_URL_CREDENTIALS, variable: 'WEBHOOK_URL')]) {
                    sh """
                        curl -X POST -H "Content-Type: application/json" \\
                        -d '{
                            "status": "UNSTABLE",
                            "project": "${JOB_NAME}",
                            "buildNumber": "${BUILD_NUMBER}",
                            "buildUrl": "${BUILD_URL}",
                            "testsStatus": "FAILED"
                        }' \\
                        '${WEBHOOK_URL}'
                    """
                }
            }
        }
    }
}