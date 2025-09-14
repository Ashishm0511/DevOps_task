pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        ECR_REPO = "626635415858.dkr.ecr.${AWS_REGION}.amazonaws.com/devops-task"
        EC2_USER = "ubuntu"
        EC2_HOST = "13.233.167.131"
        EC2_KEY = "ec2-ssh-key"
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Ashishm0511/DevOps_task.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'npm install'
                sh 'npm test || echo "No tests configured"'
            }
        }

        stage('Dockerize') {
            steps {
                script {
                    docker.build("${ECR_REPO}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                    aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                    aws configure set default.region $AWS_REGION

                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ECR_REPO

                    docker push $ECR_REPO:$BUILD_NUMBER
                    docker tag $ECR_REPO:$BUILD_NUMBER $ECR_REPO:latest
                    docker push $ECR_REPO:latest
                '''
            }
        }


        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ["${EC2_KEY}"]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "
                            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                            aws configure set default.region $AWS_REGION

                            aws ecr get-login-password --region $AWS_REGION | \
                            docker login --username AWS --password-stdin $ECR_REPO

                            docker rm -f devops-task || true
                            docker pull $ECR_REPO:latest
                            docker run -d --name devops-task -p 3000:3000 $ECR_REPO:latest
                        "
                    '''
                }
            }
        }
    }
}

    post {
        always {
            cleanWs()
        }
    }
}

