pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
        AWS_REGION = 'us-east-1'
        CLUSTER_NAME = 'my-web-cluster'
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
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        echo "Updating kubeconfig for EKS..."
                        aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
                        
                        echo "Updating image tag in deployment manifest..."
                        sed -i "s|felix081/web_app:\\${IMAGE_TAG}|felix081/web_app:${IMAGE_TAG}|g" deployment.yaml
                        
                        echo "Applying Kubernetes manifests..."
                        kubectl apply -f deployment.yaml
                        kubectl apply -f service.yaml

                        echo "Checking deployment rollout status..."
                        kubectl rollout status deployment/web-app
                        
                        echo "Getting service details..."
                        kubectl get svc web-app-service -o wide
                    '''
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: '*.yaml', allowEmptyArchive: true
            archiveArtifacts artifacts: '*-report.html', allowEmptyArchive: true
            cleanWs()
        }
        success {
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
