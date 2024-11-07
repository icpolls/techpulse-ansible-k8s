pipeline {
    agent { label 'agent1' }  // Replace 'wsl-agent' with the label of your agent

    environment {
        GITHUB_REPO_URL = 'https://github.com/vlonje20/init-app.git'
        BRANCH_NAME = 'main'  // Replace with your branch name if it's not 'main'
        GITHUB_CREDENTIALS_ID = 'vin_github_cred'  // Replace with your Jenkins GitHub credentials ID
        DOCKERHUB_CREDENTIALS_ID = 'jenkins-docker-cred'  // Replace with your Jenkins Docker Hub credentials ID
        DOCKERHUB_REPO = 'vlonje20/devops-init-app'  // Replace with your Docker Hub repository
    }

    stages {
        stage('Agent Details') {
            steps {
                echo "Running on agent: ${env.NODE_NAME}"
                sh 'uname -a'  // Print system information
                sh 'whoami'    // Print the current user
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: "${env.BRANCH_NAME}", url: "${env.GITHUB_REPO_URL}", credentialsId: "${env.GITHUB_CREDENTIALS_ID}"
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'  // Simple Maven build
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    sh 'docker --version'  // Verify Docker installation
                    sh "docker build -t ${env.DOCKERHUB_REPO}:latest ."  // Build Docker image
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${env.DOCKERHUB_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD'
                        sh "docker push ${env.DOCKERHUB_REPO}:latest"
                        sh 'docker logout'
                    }
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                script {
                    sh "docker run --name dev-init-app --rm -d -p 8383:8080 ${env.DOCKERHUB_REPO}:latest"  // Run Docker container in detached mode
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up Docker containers and images...'
            sh 'docker rm $(docker ps -a -q) || true'
            sh 'docker rmi $(docker images -q) || true'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}