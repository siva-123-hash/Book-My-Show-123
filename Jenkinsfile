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
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=BMS \
                        -Dsonar.projectKey=BMS \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Wait for SonarQube analysis result
                    def qg = waitForQualityGate abortPipeline: true
                    echo "Quality Gate Status: ${qg.status}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    cd bookmyshow-app
                    echo "Verifying package.json..."
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

        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt || true'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                            echo "Building Docker image..."
                            docker build --no-cache -t siva0927/bms:latest -f bookmyshow-app/Dockerfile bookmyshow-app

                            echo "Pushing Docker image..."
                            docker push siva0927/bms:latest
                        '''
                    }
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                sh '''
                    echo "Stopping old container..."
                    docker stop bms || true
                    docker rm bms || true

                    echo "Running new container..."
                    docker run -d --restart=always --name bms -p 3000:3000 siva0927/bms:latest

                    echo "Active containers:"
                    docker ps -a

                    echo "Container logs (5s delay)..."
                    sleep 5
                    docker logs bms
                '''
            }
        }
    }

    post {
        always {
            emailext(
                attachLog: true,
                subject: "${currentBuild.result} - ${env.JOB_NAME}",
                body: """
                    <b>Project:</b> ${env.JOB_NAME}<br/>
                    <b>Build Number:</b> ${env.BUILD_NUMBER}<br/>
                    <b>Result:</b> ${currentBuild.result}<br/>
                    <b>URL:</b> <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a><br/>
                """,
                to: 'kastrokiran@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
