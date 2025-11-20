pipeline {
    agent any

    environment {
        AWS_REGION      = "ap-south-1"
        AWS_ACCOUNT_ID  = "476400770837"
        ECR_REPO_NAME   = "myapp-repo"
        IMAGE_NAME      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        APP_SERVER_HOST = "ubuntu@13.233.159.102"   // your app EC2 public IP
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/hemanthsgit/aws-jenkins-cicd-demo.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh '''
                    echo "Installing dependencies..."
                    npm install
                    echo "Running tests..."
                    npm test
                '''
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    def commitId = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.IMAGE_TAG = commitId
                }
                sh '''
                    echo "Logging in to ECR..."
                    aws ecr get-login-password --region ${AWS_REGION} \
                        | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    echo "Building Docker image..."
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

                    echo "Tagging image as latest..."
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    echo "Pushing images to ECR..."
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent (credentials: ['app-server-key']) {
                    sh '''
                        echo "Deploying on EC2..."

                        ssh -o StrictHostKeyChecking=no ${APP_SERVER_HOST} '
                            aws ecr get-login-password --region ${AWS_REGION} \
                                | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com &&

                            docker pull ${IMAGE_NAME}:latest &&

                            (docker stop myapp || true) &&
                            (docker rm myapp || true) &&

                            docker run -d --name myapp -p 80:3000 ${IMAGE_NAME}:latest
                        '
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
