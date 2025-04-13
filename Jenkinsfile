pipeline {
    agent { label 'Agent1' }
    environment {
        GITHUB_REPO_URL = 'https://github.com/anebota/techpulse.git'
        BRANCH_NAME = 'main'
        GITHUB_CREDENTIALS_ID = 'jenkins-github-creds'
        DOCKERHUB_CREDENTIALS_ID = 'jenkins-dockerhub-creds'
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
                sh 'mvn clean package'  // Fixed: removed duplicate 'sh'
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    sh 'docker --version'
                    sh "docker build -t ${env.DOCKERHUB_REPO}:latest ."
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
                    // Add container cleanup before starting a new one to avoid conflicts
                    sh 'docker stop dev-init-app || true'
                    sh 'docker rm dev-init-app || true'
                    sh "docker run --name dev-init-app --rm -d -p 8383:8080 ${env.DOCKERHUB_REPO}:latest"
                }
            }
        }
    }
    post {
        always {
            echo 'Cleaning up Docker containers and images...'
            // Safer cleanup commands that won't fail if no containers/images exist
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
