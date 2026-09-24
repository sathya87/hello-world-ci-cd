pipeline {
    agent any

    // Local Jenkins has no public URL for a GitHub webhook, so it polls instead.
    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        // Edit these two for your EC2 box, or move them to Jenkins job parameters.
        EC2_HOST = 'ec2-user@YOUR_EC2_PUBLIC_IP'
        REMOTE_DIR = '/home/ec2-user/hello-world-ci-cd'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_HOST} '
                            cd ${REMOTE_DIR} &&
                            git pull origin main &&
                            docker compose up -d --build
                        '
                    """
                }
            }
        }
    }
}
