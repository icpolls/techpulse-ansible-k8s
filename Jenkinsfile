pipeline {
    agent { label 'mss' }
    environment {
        GITHUB_REPO_URL = 'https://github.com/anebota/techpulse.git'
        BRANCH_NAME = 'main'
        GITHUB_CREDENTIALS_ID = 'jenkins-github-creds'
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-PAT-creds'
        DOCKERHUB_REPO = 'anebota/jenkins-job-repo'
        KUBECONFIG = '/home/jenkins/.kube/config'
        ANSIBLE_PLAYBOOK = 'deploy-k8s.yml'
        SLACK_WEBHOOK = credentials('slack-webhook')
        EMAIL_RECIPIENT = credentials('notification-email')
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
                        sh 'echo $DOCKER_PAT | docker login -u $DOCKER_USERNAME --password-stdin'
                        sh "docker push ${env.DOCKERHUB_REPO}:latest"
                        sh 'docker logout'
                    }
                }
            }
        }
        stage('Run Docker Container Locally') {
            steps {
                script {
                    sh 'docker stop ms-app || true'
                    sh 'docker rm ms-app || true'
                    sh "docker run --name ms-app --rm -d -p 8282:8080 ${env.DOCKERHUB_REPO}:latest"
                }
            }
        }
        stage('Run Ansible Playbook') {
            steps {
                echo "Executing Ansible playbook for Kubernetes deployment..."
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    sh "ANSIBLE_CONFIG=/etc/ansible/ansible.cfg ansible-playbook -i inventory.ini $ANSIBLE_PLAYBOOK"
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                echo "Deploying application to Kubernetes..."
                sh "kubectl apply -f k8s/deployment.yml --kubeconfig=${env.KUBECONFIG}"
            }
        }
        stage('Enable Autoscaling') {
            steps {
                echo "Creating Horizontal Pod Autoscaler to scale by 1 pod on 80% CPU..."
                sh "kubectl autoscale deployment techpulse-deployment --cpu-percent=80 --min=1 --max=2 --kubeconfig=${env.KUBECONFIG}"
            }
        }
        stage('Verify Deployment') {
            steps {
                echo "Verifying deployment status..."
                sh "kubectl get pods --kubeconfig=${env.KUBECONFIG}"
                sh "kubectl get hpa --kubeconfig=${env.KUBECONFIG}"
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
            echo 'Pipeline executed successfully!'
            slackSend(color: '#36a64f', message: ":rocket: SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open Build>)")
            mail to: "${env.EMAIL_RECIPIENT}", subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}", body: "Pipeline succeeded.\nCheck details: ${env.BUILD_URL}"
        }
        failure {
            echo 'Pipeline failed!'
            slackSend(color: '#FF0000', message: ":x: FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open Build>)")
            mail to: "${env.EMAIL_RECIPIENT}", subject: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}", body: "Pipeline failed.\nCheck details: ${env.BUILD_URL}"
        }
    }
}
