pipeline {
    agent any
    stages {
        stage('Preparation') {
            steps {
                script {
                    slackSend(channel: 'devops', message: "Starting the pipeline for ${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER}")
                }
                // Checkout the repository from GitHub
                git(
                    url: 'https://github.com/mostafa-7ussein/semicolonProject',
                    branch: 'main'
                )
            }
        }
        stage('Test') {
            steps {
                echo "Running Docker Compose tests"
                // Clean up any previous instances and start new ones
                sh "docker compose -f docker-compose-testing.yml down --remove-orphans"
                sh "docker compose -f docker-compose-testing.yml up -d --build"
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    // Define the image name and tag using Git commit hash
                    def imageTag = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    def imageName = "mostafahu/semicolon-backend:${imageTag}"
                    
                    // Build the Docker image
                    sh "docker build . -t ${imageName}"

                    // Check if the image already exists on Docker Hub
                    def imageExists = sh(script: "docker manifest inspect ${imageName}", returnStatus: true)

                    // Use Jenkins credentials for Docker Hub login
                    withCredentials([usernamePassword(credentialsId: 'DockerHub', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                        if (imageExists != 0) {
                            // Log in to Docker Hub
                            sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'

                            // Push Docker image to Docker Hub
                            sh "docker push ${imageName}"

                            // Tag as 'latest' and push
                            sh "docker tag ${imageName} mostafahu/semicolon-backend:latest"
                            sh "docker push mostafahu/semicolon-backend:latest"

                            echo "Docker image ${imageName} pushed successfully."
                        } else {
                            echo "No changes detected, Docker image ${imageName} already exists. Skipping push."
                        }
                    }
                }
            }
        }
        stage('Deploy to Production') {
            steps {
                script {
                    // Call Ansible playbook to deploy the production environment
                    sh "ansible-playbook -i inventory/production playbook.yaml --ssh-extra-args='-o StrictHostKeyChecking=no'"
                }
            }
        }
    }
    post {
        success {
            script {
                slackSend(channel: '#devops',color: '#00FF00', message: "Succeeded  ${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER} succeeded!")
            }
        }
        failure {
            script {
                slackSend(channel: '#devops', message: "Pipeline ${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER} failed!")
            }
        }
        always {
            script {
                slackSend(channel: '#devops', color: '#FF0000', message: "Failed: ${env.JOB_NAME} - Build Number: ${env.BUILD_NUMBER} finished.")
            }
        }
    }
}

