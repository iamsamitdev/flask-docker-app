pipeline {
    // ใช้ any agent เพื่อหลีกเลี่ยงปัญหา Docker path mounting บน Windows
    agent any

    // กำหนด environment variables
    environment {
        // ใช้ค่าเป็น "credentialsId" ของ Jenkins โดยตรงสำหรับ docker.withRegistry
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub-cred'
        DOCKER_REPO = "iamsamitdev/flask-docker-app"
        APP_NAME = "flask-docker-app"
    }

    // กำหนด stages ของ Pipeline
    stages {
        // Stage 1: ดึงโค้ดล่าสุดจาก Git
        stage('Checkout') {
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        // Stage 2: ติดตั้ง dependencies และรันเทสต์ (รองรับทุก Platform)
        stage('Install & Test') {
            steps {
                script {
                    // ตรวจสอบว่ามี Python บน host หรือไม่ และรันด้วย host ถ้ามี ไม่เช่นนั้น fallback เป็น Docker
                    def hasPython = false
                    def isWindows = isUnix() ? false : true

                    try {
                        if (isWindows) {
                            bat 'python --version && pip --version'
                        } else {
                            sh 'python3 --version && python3 -m pip --version'
                        }
                        hasPython = true
                        echo "Using Python installed on ${isWindows ? 'Windows' : 'Unix'}"
                    } catch (Exception e) {
                        echo "Python not found on host, using Docker"
                        hasPython = false
                    }

                    if (hasPython) {
                        // ใช้ Python บน host
                        if (isWindows) {
                            bat '''
                                python -m pip install --upgrade pip
                                pip install -r requirements.txt
                                pytest -v --tb=short --junitxml=test-results.xml
                            '''
                        } else {
                            sh '''
                                python3 -m pip install --upgrade pip
                                python3 -m pip install -r requirements.txt
                                python3 -m pytest tests/ -v --tb=short --junitxml=test-results.xml
                            '''
                        }
                    } else {
                        // ใช้ Docker run command (รองรับทุก platform)
                        if (isWindows) {
                            bat '''
                                docker run --rm ^
                                -v "%cd%":/workspace ^
                                -w /workspace ^
                                python:3.13-slim sh -c "pip install -r requirements.txt && pytest -v --tb=short --junitxml=test-results.xml"
                            '''
                        } else {
                            sh '''
                                docker run --rm \
                                -v "$(pwd)":/workspace \
                                -w /workspace \
                                python:3.13-slim sh -c "pip install -r requirements.txt && pytest -v --tb=short --junitxml=test-results.xml"
                            '''
                        }
                    }
                }
            }
            post {
                always {
                    // บันทึกผลการทดสอบเป็น JUnit เพื่อแสดงใน Jenkins
                    junit 'test-results.xml'
                }
            }
        }

        // Stage 3: สร้าง Docker Image สำหรับ production
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_REPO}:${BUILD_NUMBER}"
                    docker.build("${DOCKER_REPO}:${BUILD_NUMBER}", ".")
                }
            }
        }

        // Stage 4: Push Image ไปยัง Docker Hub
        stage('Push Docker Image') {
            steps {
                script {
                    // ต้องส่งค่าเป็น credentialsId เท่านั้น ไม่ใช่ค่าที่ mask ของ credentials()
                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_HUB_CREDENTIALS_ID) {
                        echo "Pushing image to Docker Hub..."
                        def image = docker.image("${DOCKER_REPO}:${BUILD_NUMBER}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        // Stage 5: Cleanup local Docker cache/images on the agent to save disk space
        stage('Cleanup Docker') {
            steps {
                script {
                    def isWindows = isUnix() ? false : true
                    echo "Cleaning up local Docker images/cache on agent..."
                    if (isWindows) {
                        bat """
                            docker image rm -f ${DOCKER_REPO}:${BUILD_NUMBER} || echo ignore
                            docker image rm -f ${DOCKER_REPO}:latest || echo ignore
                            docker image prune -af -f
                            docker builder prune -af -f
                        """
                    } else {
                        sh """
                            docker image rm -f ${DOCKER_REPO}:${BUILD_NUMBER} || true
                            docker image rm -f ${DOCKER_REPO}:latest || true
                            docker image prune -af -f
                            docker builder prune -af -f
                        """
                    }
                }
            }
        }

        // Stage 6: Deploy latest image to localhost for smoke test
        stage('Deploy Local') {
            steps {
                script {
                    def isWindows = isUnix() ? false : true
                    echo "Deploying container ${APP_NAME} from latest image..."
                    if (isWindows) {
                        bat """
                            docker pull ${DOCKER_REPO}:latest
                            docker stop ${APP_NAME} || echo ignore
                            docker rm ${APP_NAME} || echo ignore
                            docker run -d --name ${APP_NAME} -p 5000:5000 ${DOCKER_REPO}:latest
                            docker ps --filter name=${APP_NAME} --format \"table {{.Names}}\t{{.Image}}\t{{.Status}}\"
                        """
                    } else {
                        sh """
                            docker pull ${DOCKER_REPO}:latest
                            docker stop ${APP_NAME} || true
                            docker rm ${APP_NAME} || true
                            docker run -d --name ${APP_NAME} -p 5000:5000 ${DOCKER_REPO}:latest
                            docker ps --filter name=${APP_NAME} --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
                        """
                    }
                }
            }
            post {
                success {
                    script {
                        def isWindows = isUnix() ? false : true
                        withCredentials([string(credentialsId: 'n8n-webhook', variable: 'N8N_WEBHOOK_URL')]) {
                            if (isWindows) {
                                // ใช้ PowerShell แบบบรรทัดเดียว (ไม่มี caret ^) เพื่อหลีกเลี่ยง error ใน cmd
                                bat '''
                                    powershell -NoProfile -Command "$body = [PSCustomObject]@{ project=$env:JOB_NAME; stage='Deploy Local'; status='success'; build=$env:BUILD_NUMBER; image=($env:DOCKER_REPO + ':latest'); container=$env:APP_NAME; url='http://localhost:5000/'; timestamp=(Get-Date -Format o) }; $json = $body | ConvertTo-Json; Invoke-RestMethod -Uri $env:N8N_WEBHOOK_URL -Method Post -ContentType 'application/json' -Body $json"
                                '''
                            } else {
                                sh """
                                    curl -s -X POST "$N8N_WEBHOOK_URL" \
                                      -H 'Content-Type: application/json' \
                                      -d '{
                                            "project": "${JOB_NAME}",
                                            "stage": "Deploy Local",
                                            "status": "success",
                                            "build": "${BUILD_NUMBER}",
                                            "image": "${DOCKER_REPO}:latest",
                                            "container": "${APP_NAME}",
                                            "url": "http://localhost:5000/"
                                          }'
                                """
                            }
                        }
                    }
                }
            }
        }

        // Stage 7: Deploy to remote server (optional)
        // stage('Deploy to Server') {
        //     steps {
        //         sshagent(['deploy-server-cred']) {
        //             sh """
        //                 ssh -o StrictHostKeyChecking=no $DEPLOY_SERVER '
        //                     docker pull ${DOCKER_REPO}:latest &&
        //                     docker stop ${APP_NAME} || true &&
        //                     docker rm ${APP_NAME} || true &&
        //                     docker run -d --name ${APP_NAME} -p 5000:5000 ${DOCKER_REPO}:latest
        //                 '
        //             """
        //         }
        //     }

        //     post {
        //         success {
        //             script {
        //                 def isWindows = isUnix() ? false : true
        //                 withCredentials([string(credentialsId: 'n8n-webhook', variable: 'N8N_WEBHOOK_URL')]) {
        //                     if (isWindows) {
                                // ใช้ PowerShell แบบบรรทัดเดียว (ไม่มี caret ^) เพื่อหลีกเลี่ยง error ใน cmd
        //                         bat '''
        //                             powershell -NoProfile -Command "$body = [PSCustomObject]@{ project=$env:JOB_NAME; stage='Deploy Local'; status='success'; build=$env:BUILD_NUMBER; image=($env:DOCKER_REPO + ':latest'); container=$env:APP_NAME; url='http://localhost:5000/'; timestamp=(Get-Date -Format o) }; $json = $body | ConvertTo-Json; Invoke-RestMethod -Uri $env:N8N_WEBHOOK_URL -Method Post -ContentType 'application/json' -Body $json"
        //                         '''
        //                     } else {
        //                         sh """
        //                             curl -s -X POST "$N8N_WEBHOOK_URL" \
        //                               -H 'Content-Type: application/json' \
        //                               -d '{
        //                                     "project": "${JOB_NAME}",
        //                                     "stage": "Deploy Local",
        //                                     "status": "success",
        //                                     "build": "${BUILD_NUMBER}",
        //                                     "image": "${DOCKER_REPO}:latest",
        //                                     "container": "${APP_NAME}",
        //                                     "url": "http://localhost:5000/"
        //                                   }'
        //                         """
        //                     }
        //                 }
        //             }
        //         }
        //     }
        // }
    }

    // แจ้งเตือนเมื่อ Pipeline ล้มเหลว (failure)
    post {
        failure {
            script {
                def isWindows = isUnix() ? false : true
                withCredentials([string(credentialsId: 'n8n-webhook', variable: 'N8N_WEBHOOK_URL')]) {
                    if (isWindows) {
                        bat '''
                            powershell -NoProfile -Command "$body = [PSCustomObject]@{ project=$env:JOB_NAME; stage='Pipeline'; status='failed'; build=$env:BUILD_NUMBER; image=($env:DOCKER_REPO + ':latest'); container=$env:APP_NAME; url='http://localhost:5000/'; timestamp=(Get-Date -Format o) }; $json = $body | ConvertTo-Json; Invoke-RestMethod -Uri $env:N8N_WEBHOOK_URL -Method Post -ContentType 'application/json' -Body $json"
                        '''
                    } else {
                        sh """
                            curl -s -X POST "$N8N_WEBHOOK_URL" \
                              -H 'Content-Type: application/json' \
                              -d '{
                                    "project": "${JOB_NAME}",
                                    "stage": "Pipeline",
                                    "status": "failed",
                                    "build": "${BUILD_NUMBER}",
                                    "image": "${DOCKER_REPO}:latest",
                                    "container": "${APP_NAME}",
                                    "url": "http://localhost:5000/"
                                  }'
                        """
                    }
                }
            }
        }
    }
}