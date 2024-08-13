pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'Sonar-scanner'
        SONAR_TOKEN = credentials('sonar-token')
    }

    stages {
        stage('Start SonarQube') {
            steps {
                script {
                    echo "Checking if SonarQube is running..."
                    def sonarRunning = sh(script: 'docker ps -q -f name=sonar', returnStatus: true) == 0
                    if (!sonarRunning) {
                        echo "SonarQube is not running. Starting SonarQube..."
                        sh 'docker start sonar || docker run -d --name sonar -p 9000:9000 sonarqube'
                    } else {
                        echo "SonarQube is already running."
                    }
                }
            }
        }

        stage('Git Checkout') {
            steps {
                echo "Checking out the git repository..."
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Ademnajjar/microservices.git'
            }
        }

        stage('Compile') {
            steps {
                echo "Compiling the project..."
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                echo "Running tests..."
                sh "mvn test"
            }
        }

        stage('Check Trivy Installation') {
            steps {
                script {
                    echo "Checking if Trivy is installed..."
                    def trivyInstalled = sh(script: 'command -v trivy', returnStatus: true) == 0
                    if (!trivyInstalled) {
                        error "Trivy is not installed on this Jenkins agent."
                    } else {
                        echo "Trivy is installed."
                    }
                }
            }
        }

        stage('File System Scan') {
            steps {
                echo "Running Trivy file system scan..."
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube analysis..."
                withSonarQubeEnv('sonar') {
                    sh """${SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectName=microservices \
                          -Dsonar.projectKey=microservices \
                          -Dsonar.login=${SONAR_TOKEN} \
                          -Dsonar.java.binaries=. \
                          -X"""
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    echo "Waiting for SonarQube quality gate..."
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        echo "Quality gate failed: ${qg.status}. Proceeding with the pipeline."
                        currentBuild.result = 'UNSTABLE'
                    } else {
                        echo "Quality gate passed: ${qg.status}"
                    }
                }
            }
        }

        stage('Build') {
            steps {
                echo "Building the project..."
                sh "mvn package"
            }
        }

        stage('Publish To Nexus') {
            steps {
                echo "Publishing to Nexus..."
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -X"
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    echo "Building and tagging Docker image..."
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t Ademnajjar/microservices:latest ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                echo "Running Trivy image scan..."
                sh "trivy image --format table -o trivy-image-report.html Ademnajjar/microservices:latest"
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo "Pushing Docker image to registry..."
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push Ademnajjar/microservices:latest"
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                echo "Deploying to Kubernetes..."
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.8.146:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                echo "Verifying the deployment in Kubernetes..."
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.8.146:6443') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """
            }
        }
    }
}
