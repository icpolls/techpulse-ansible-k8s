pipeline {
    agent { label 'mss' }
    environment {
        GITHUB_REPO_URL = 'https://github.com/anebota/techpulse.git'
        BRANCH_NAME = 'main'
        GITHUB_CREDENTIALS_ID = 'jenkins-github-creds'
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-PAT-creds'
        DOCKERHUB_REPO = 'anebota/jenkins-job-repo'
    }
    stages {
        stage('Agent Details') {
            steps {
                echo "Running on agent: ${env.NODE_NAME}"
                sh 'uname -a'
                sh 'whoami'
            }
        }
        stage('Clone Repository') {
            steps {
                git branch: "${env.BRANCH_NAME}", url: "${env.GITHUB_REPO_URL}", credentialsId: "${env.GITHUB_CREDENTIALS_ID}"
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    sh 'docker --version'
                    // Echo the repo name to verify it's correctly formatted
                    echo "Building Docker image: ${env.DOCKERHUB_REPO}:latest"
                    sh "docker build -t ${env.DOCKERHUB_REPO}:latest ."
                }
            }
        }
        stage('Docker Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${env.DOCKERHUB_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PAT')]) {
                        echo "Logging into Docker Hub as ${DOCKER_USERNAME}"
                        // Using the newer recommended authentication approach
                        sh 'echo $DOCKER_PAT | docker login -u $DOCKER_USERNAME --password-stdin'
                        sh "docker push ${env.DOCKERHUB_REPO}:latest"
                        sh 'docker logout'
                    }
                }
            }
        }
        stage('Run Docker Container') {
            steps {
                script {
                    sh 'docker stop ms-app || true'
                    sh 'docker rm ms-app || true'
                    sh "docker run --name ms-app --rm -d -p 8383:8080 ${env.DOCKERHUB_REPO}:latest"
                }
            }
        }
    }
    post {
        always {
            echo 'Cleaning up Docker containers and images...'
            sh 'docker container prune -f || true'
            sh 'docker image prune -af || true'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
