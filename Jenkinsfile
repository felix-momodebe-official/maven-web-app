pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${env.BUILD_NUMBER}" // Corrected IMAGE_TAG reference
        AWS_ACCESS_KEY_ID = credentials('aws-cred')
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

        stage('Trivy FS Security Scan') {
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
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

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
        }

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
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    caCertificate: '', 
                    clusterName: 'devopsola-cluster', 
                    contextName: '', 
                    credentialsId: 'kubeconfig-secrete', 
                    namespace: '', 
                    restrictKubeConfigAccess: false, 
                    serverUrl: 'https://39FC4F671ACD6A5FD62614BB82E7E580.gr7.us-east-1.eks.amazonaws.com'
                ) {
                    sh 'kubectl --kubeconfig=$KUBECONFIG get nodes' // Added validation step
                    sh 'kubectl --kubeconfig=$KUBECONFIG apply -f deployment.yaml'
                    sh 'kubectl --kubeconfig=$KUBECONFIG apply -f service.yaml'
                    sh 'kubectl --kubeconfig=$KUBECONFIG rollout status deployment/web-app' // Verify successful rollout
                }
            }
        }
    }
}
