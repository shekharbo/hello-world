pipeline {
    agent any
    environment {
        // Define environment variables
        SSH_KEY = credentials('ssh_key')
        EC2_IP = '3.88.55.152'
    }
    stages {
        stage('Checkout') {
            steps {
                // Checkout code from Git repository
                git branch: 'main', url: 'https://github.com/shekharbo/hello-world.git'
            }
        }
        stage('Deploy') {
            steps {
                script {
                    // SSH into EC2 instance and deploy the application
                    sh '''
                    ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ubuntu@${EC2_IP} << EOF
                    sudo apt update -y
                    sudo apt install -y python3 git
                    git clone https://github.com/shekharbo/hello-world.git
                    cd /home/ubuntu/hello-world/hello-world
                    python3 main.py &
                    exit
                    EOF
                    '''
                }
            }
        }
    }
}

