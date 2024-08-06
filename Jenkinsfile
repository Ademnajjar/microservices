pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'Sonar-scanner'
        SONAR_TOKEN = 'squ_f7f3b3d80d89103651bc8e0cca98eff378b893a3'
    }

    stages {
        stage('Start SonarQube') {
            steps {
                script {
                    def sonarRunning = sh(script: 'docker ps -q -f name=sonar', returnStatus: true) == 0
                    if (!sonarRunning) {
                        sh 'docker start sonar || docker run -d --name sonar -p 9000:9000 sonarqube'
                    }
                }
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Ademnajjar/microservices.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('Check Trivy Installation') {
            steps {
                script {
                    def trivyInstalled = sh(script: 'command -v trivy', returnStatus: true) == 0
                    if (!trivyInstalled) {
                        error "Trivy is not installed on this Jenkins agent."
                    }
                }
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """${SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectName=micro
