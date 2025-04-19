pipeline {
    agent any

    environment {
        DOCKER_USERNAME = credentials('DOCKER_USERNAME')
        DOCKER_PASSWORD = credentials('DOCKER_PASSWORD')
        MONGO_URI = credentials('MONGO_URI')
    }

    stages {
        stage('Checkout code') {
            steps {
                git branch: 'main', credentialsId: 'Github', url: 'https://github.com/harshkr-cs/BookStore.git'
            }
        }

        stage('Log in to Docker Hub') {
            steps {
                script {
                    // NOTE: Avoid piping since it's not supported on Windows in the same way
                    bat """
                        echo Logging in to Docker Hub
                        docker login -u %DOCKER_USERNAME% -p %DOCKER_PASSWORD%
                    """
                }
            }
        }

        stage('Build and Push Server Image') {
            steps {
                script {
                    bat """
                        docker build -t %DOCKER_USERNAME%/backend:latest ./Backend
                        docker push %DOCKER_USERNAME%/backend:latest
                    """
                }
            }
        }

        stage('Upload docker-compose.yml to EC2') {
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'EC2_SSH',
                        transfers: [
                            sshTransfer(
                                sourceFiles: 'docker-compose.yml',
                                remoteDirectory: 'Bookstore',
                                removePrefix: '',
                                execCommand: '',
                                execTimeout: 120000
                            )
                        ],
                        usePromotionTimestamp: false,
                        verbose: true
                    )
                ])
            }
        }

        stage('Deploy to EC2 machine') {
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'EC2_SSH',
                        transfers: [
                            sshTransfer(
                                execCommand: """
                                    cd Bookstore &&
                                    export DOCKER_USERNAME=${DOCKER_USERNAME} &&
                                    export MONGO_URI=${MONGO_URI} &&

                                    docker-compose down &&
                                    docker rmi ${DOCKER_USERNAME}/backend:latest || true &&
                                    docker system prune -af &&
                                    docker-compose pull &&
                                    docker-compose up -d --force-recreate
                                """,
                                execTimeout: 120000
                            )
                        ],
                        usePromotionTimestamp: false,
                        verbose: true
                    )
                ])
            }
        }
    }

    post {
        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
    }
}
