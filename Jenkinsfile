pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/siva-123-hash/Book-My-Show-123.git'
                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    retry(2) { // ✅ Retry if interrupted or SonarQube fails
                        withSonarQubeEnv('sonar-server') {
                            sh '''
                                echo "Starting SonarQube Analysis..."
                                $SCANNER_HOME/bin/sonar-scanner \
                                    -Dsonar.projectKey=BMS \
                                    -Dsonar.projectName=BMS \
                                    -Dsonar.projectVersion=1.0 \
                                    -Dsonar.sources=. \
                                    -Dsonar.language=js \
                                    -Dsonar.sourceEncoding=UTF-8 \
                                    -Dsonar.java.binaries=target \
                                    -Dsonar.qualitygate.wait=true
                            '''
                            echo "SonarQube analysis completed."
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // ✅ Wait for SonarQube Quality Gate result (up to 5 minutes)
                    timeout(time: 5, unit: 'MINUTES') {
                        def qg = waitForQualityGate abortPipeline: true
                        echo "Quality Gate Status: ${qg.status}"
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    cd bookmyshow-app
                    ls -la
                    if [ -f package.json ]; then
                        rm -rf node_modules package-lock.json
                        npm install
                    else
                        echo "Error: package.json not found in bookmyshow-app!"
                        exit 1
                    fi
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                            echo "Building Docker image..."
                            docker build -t siva0927/bms:latest -f bookmyshow-app/Dockerfile bookmyshow-app

                            echo "Pushing Docker image to registry..."
                            docker push siva0927/bms:latest
                        '''
                    }
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                sh '''
                    echo "Stopping and removing old container..."
                    docker stop bms || true
                    docker rm bms || true

                    echo "Running new container on port 3000..."
                    docker run -d --restart=always --name bms -p 3000:3000 siva0927/bms:latest

                    echo "Checking running containers..."
                    docker ps -a

                    echo "Fetching logs..."
                    sleep 5
                    docker logs bms
                '''
            }
        }
    }
}
