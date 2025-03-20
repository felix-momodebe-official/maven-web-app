pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                checkout scmGit(branches: [[name: 'main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/felix-momodebe-official/maven-web-app.git']])
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
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
                          -Dsonar.projectKey=web-app \
                          -Dsonar.java.binaries=target '''
                }
            }
        }

        stage('SonarQube Quality GateCheck') {
            steps {
                timeout(5) {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Publish Artifact To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-web', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }  // <-- Fixed missing closing brace

        stage('Docker Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker build -t felix081/web-app:$IMAGE_TAG .'
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --format table -o image-report.html felix081/web-app:$IMAGE_TAG'
            }
        }

        stage('Docker Push') {  // <-- Fixed duplicate stage name
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker push felix081/web-app:$IMAGE_TAG'
                    }
                }
            }
        }
    }  // <-- Fixed missing closing brace for "stages"
}  // <-- Fixed missing closing brace for "pipeline"
