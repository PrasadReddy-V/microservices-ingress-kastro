pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ACCOUNT_ID = '138094353328'   // 🔴 change if needed
        ECR_REPO = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/techsolutions-app"

        K8S_CLUSTER_NAME = 'prasad-cluster'
        NAMESPACE = 'default'
        APP_NAME = 'techsolutions'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                git 'https://github.com/PrasadReddy-V/microservices-ingress-kastro.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    def buildNumber = env.BUILD_NUMBER

                    sh "docker build -t techsolutions-app:${buildNumber} ."
                    sh "docker tag techsolutions-app:${buildNumber} ${ECR_REPO}:${buildNumber}"
                    sh "docker tag techsolutions-app:${buildNumber} ${ECR_REPO}:latest"

                    env.IMAGE_TAG = buildNumber
                }
            }
        }

        stage('Login to ECR') {
            steps {
                echo 'Logging into ECR...'
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                        sh """
                        aws ecr get-login-password --region ${AWS_REGION} \
                        | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        """
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                echo 'Pushing Docker image to ECR...'
                script {
                    sh "docker push ${ECR_REPO}:${env.IMAGE_TAG}"
                    sh "docker push ${ECR_REPO}:latest"
                }
            }
        }

        stage('Configure AWS and Kubectl') {
            steps {
                echo 'Configuring AWS CLI and kubectl...'
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                        sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${K8S_CLUSTER_NAME}"
                        sh "kubectl get nodes"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying application to Kubernetes...'
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {

                        sh """
                        sed -i 's|image: .*|image: ${ECR_REPO}:${env.IMAGE_TAG}|g' k8s/deployment.yaml
                        """

                        sh "kubectl apply -f k8s/deployment.yaml"
                        sh "kubectl rollout status deployment/${APP_NAME}-deployment --timeout=300s"
                    }
                }
            }
        }

        stage('Deploy Ingress') {
            steps {
                echo 'Deploying Ingress resource...'
                script {
                    sh "kubectl apply -f k8s/ingress.yaml"
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up Docker images...'
            sh "docker rmi ${ECR_REPO}:${env.IMAGE_TAG} || true"
            sh "docker rmi ${ECR_REPO}:latest || true"
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed! Please check the logs.'
        }
    }
}
