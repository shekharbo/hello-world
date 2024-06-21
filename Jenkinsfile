pipeline {
    agent {
        label 'ec2'
    }

    environment { 
        //AWS_REGION = "us-east-1"
        AWS_ECR_REGION = "us-east-1"
        AWS_ECR_REPOSITORY = "584294315145.dkr.ecr.us-east-1.amazonaws.com"
        DOCKER_IMAGE_NAME = "github-poc"
        AWS_ECR_ACCOUNT_ID = "58429431514"
    }

    stages {
        stage('Checkout') {
            steps {
                retry(3) {
                    checkout([$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs: [[url: 'https://github.com/shekharbo/hello-world.git']]])
                }
            }
        }

        stage('Unit Test') {
            steps {
                sh 'docker --version'
                sh 'docker pull python:3.9'
                sh 'pip install -r requirements.txt'
                sh 'python3 -m venv ~/myenv'
                sh """
                set +x
                . /home/ubuntu/myenv/bin/activate
                """
                sh """
                /home/ubuntu/.local/bin/pytest /home/ubuntu/workspace/hello_world_demo/tests/test_main.py
               """
            }
        }

        stage('Code Coverage') {
            steps {
                sh 'pip install coverage'
                sh """
                /home/ubuntu/.local/bin/coverage run -m pytest /home/ubuntu/workspace/hello_world_demo/tests/test_main.py
                """
                sh '/home/ubuntu/.local/bin/coverage report'
                sh '/home/ubuntu/.local/bin/coverage xml -o coverage.xml'
                cobertura coberturaReportFile: 'coverage.xml'
            }
        }

        stage('SCA and SonarQube') {
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    script {
                        def scannerHome = tool 'SonarQubeScanner'
                        if (scannerHome) {
                            sh "/home/hello-world-demo-python/hello-world/sonar-scanner-5.0.1.3006-linux/bin/sonar-scanner \
                                -Dsonar.projectKey=hello-world \
                                -Dsonar.sources=src \
                                -Dsonar.host.url=${SONARQUBE_SERVER} \
                                -Dsonar.login=${SONARQUBE_LOGIN_TOKEN}"
                        } else {
                            error "SonarQube Scanner not configured."
                        }
                    }
                }
            }
        }

        stage('Build and tag image using Docker') {
            steps {
                script {
                    dir('/home/ubuntu/hello-world-demo-python/hello-world') {
                        sh 'pwd'
                        sh 'ls -l Dockerfile'
                        sh 'docker build -t hello-world-app .'
                        sh "docker tag hello-world-app ${AWS_ECR_ACCOUNT_ID}.dkr.ecr.${AWS_ECR_REGION}.amazonaws.com/${AWS_ECR_REPOSITORY}:latest"
                        withCredentials([usernamePassword(credentialsId: 'aws-ecr-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                            sh "aws ecr get-login-password --region ${AWS_ECR_REGION} | docker login --username AWS --password-stdin ${AWS_ECR_ACCOUNT_ID}.dkr.ecr.${AWS_ECR_REGION}.amazonaws.com"
                            sh "docker push ${AWS_ECR_ACCOUNT_ID}.dkr.ecr.${AWS_ECR_REGION}.amazonaws.com/${AWS_ECR_REPOSITORY}:latest"
                        }
                    }
                }
            }
        }

        stage('Image scan using trivy') {
            steps {
                sh "trivy image ${AWS_ECR_ACCOUNT_ID}.dkr.ecr.${AWS_ECR_REGION}.amazonaws.com/${AWS_ECR_REPOSITORY}:latest"
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ssh_key']) {
                    sh "ssh -o StrictHostKeyChecking=no ubuntu@52.205.167.69 aws ecr get-login-password --region ${AWS_ECR_REGION} | docker login --username AWS --password-stdin ${DOCKER_REGISTRY}"
                    sh "ssh -o StrictHostKeyChecking=no ubuntu@52.205.167.69 docker pull ${AWS_ECR_ACCOUNT_ID}.dkr.ecr.${AWS_ECR_REGION}.amazonaws.com/${AWS_ECR_REPOSITORY}:latest"
                    sh "ssh -o StrictHostKeyChecking=no ubuntu@52.205.167.69 docker run -d -p 8081:8080 ${AWS_ECR_ACCOUNT_ID}.dkr.ecr.${AWS_ECR_REGION}.amazonaws.com/${AWS_ECR_REPOSITORY}:latest"
                }
            }
        }
    }
}
