# A Manual deployment Logo Server

A simple Express.js web server that serves the Swayatt logo image.

## What is this app?

This is a lightweight Node.js application built with Express.js that serves a single logo image (`logoswayatt.png`) when accessed through a web browser. When you visit the root URL, the server responds by displaying the Swayatt logo.

## Prerequisites

- Node.js (version 12 or higher)
- npm (Node Package Manager)

## Installation

1. Clone or download this repository
2. Navigate to the project directory:
   ```bash
   cd "devops task"
   ```
3. Install dependencies:
   ```bash
   npm install
   ```

## How to Start the App

Run the following command:
```bash
npm start
```

The server will start and display:
```
Server running on http://localhost:3000
```

## Usage

Once the server is running, open your web browser and navigate to:
```
http://localhost:3000
```

You will see the Swayatt logo displayed in your browser.

## Project Structure

```
├── app.js              # Main server file
├── package.json        # Project dependencies and scripts
├── logoswayatt.png     # Logo image file
└── README.md          # This file
```

## Technical Details

- **Framework**: Express.js
- **Port**: 3000
- **Endpoint**: GET `/` - serves the logo image
- **File served**: `logoswayatt.png`


# After that deployed using pipeline 

CI/CD Pipeline with Jenkins, Docker, AWS ECR & EC2

#Flow of pipelines
Checkout
Build and test
Dockerize
Push to ECR
Deploy to EC2


#Jenkins Credentials Setup

AWS Credentials

Go to Manage Jenkins → Credentials → Global → Add Credentials.
Type: AWS Credentials.
ID: aws-credentials.
EC2 SSH Key

Type: SSH Username with private key.
Username: ec2-user (or ubuntu, depending on AMI).
Private key: Paste PEM contents.
ID: ec2-ssh-key

Now create pipeline

pipeline {
    agent any
    environment {
        AWS_REGION = "ap-south-1"
        ECR_REPO = "626635415858.dkr.ecr.ap-south-1.amazonaws.com/devops-task"
        EC2_USER = "ubuntu"
        EC2_HOST = "ec2-XX-XX-XX-XX.ap-south-1.compute.amazonaws.com"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Ashishm0511/DevOps_task.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh '''
                    npm install
                    npm test || echo "No tests configured"
                '''
            }
        }

        stage('Dockerize') {
            steps {
                sh '''
                    docker build -t ${ECR_REPO}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REPO}

                        docker push ${ECR_REPO}:${BUILD_NUMBER}
                        docker tag ${ECR_REPO}:${BUILD_NUMBER} ${ECR_REPO}:latest
                        docker push ${ECR_REPO}:latest
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "
                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REPO}

                            docker rm -f devops-task || true
                            docker pull ${ECR_REPO}:latest
                            docker run -d --name devops-task -p 3000:3000 ${ECR_REPO}:latest
                        "
                    '''
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
