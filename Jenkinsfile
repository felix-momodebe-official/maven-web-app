pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${env.BUILD_NUMBER}"  // Corrected IMAGE_TAG reference
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/felix-momodebe-official/maven-web-app.git'
            }
        }
        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Trivy FS') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=web-app \
                          -Dsonar.projectKey=webapp \
                          -Dsonar.java.binaries=target'''
                }
            }
        } // Closing brace for 'SonarQube Analysis'

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        } // Closing brace for 'Quality Gate Check'

        stage('Maven Build') {
            steps {
                sh 'mvn package'
            }
        }
        
        stage('Publish Artifact') {
            steps {
                withMaven(globalMavenSettingsConfig: 'webapp', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Docker Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker build -t felix081/web_app:$IMAGE_TAG .'
                    }
                }
            }
        } // Closing brace for 'Docker Build & Tag'

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --format table -o image-report.html felix081/web_app:$IMAGE_TAG'
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker push felix081/web_app:$IMAGE_TAG'
                    }
                }
            }
        } // Closing brace for 'Push Docker Image.'
    }
}
