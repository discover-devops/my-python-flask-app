pipeline {
    agent any

    environment {
        GIT_CREDENTIALS_ID = '09c0a441-cd75-47c3-ab5e-7bd95379329d'
        DOCKER_CREDENTIALS_ID = '3fa0acbd-4512-4d55-9b54-bb74cc08278d'
        DOCKER_IMAGE = 'discoverdevops/my-python-flask-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/chrahul/my-python-flask-app.git',
                        credentialsId: env.GIT_CREDENTIALS_ID
                    ]]
                ])
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    docker.image('python:3.8').inside {
                        sh 'pip install -r requirements.txt'
                    }
                }
            }
        }

        stage('Code Linting') {
            steps {
                script {
                    docker.image('python:3.8').inside {
                        sh 'flake8 .'
                    }
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    docker.image('python:3.8').inside {
                        sh 'pytest --junitxml=reports/test-results.xml'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${env.DOCKER_IMAGE}:${env.BUILD_ID}")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', env.DOCKER_CREDENTIALS_ID) {
                        dockerImage.push("${env.BUILD_ID}")
                        dockerImage.push("latest")
                    }
                }
            }
        }
    }

    post {
        always {
            junit 'reports/test-results.xml'
        }
        success {
            echo 'Build succeeded!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
