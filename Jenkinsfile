pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/chrahul/my-python-project.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }

        stage('Code Linting') {
            steps {
                sh 'flake8 .'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'pytest'
            }
        }
    }

    post {
        always {
            junit 'reports/*.xml'
        }
        success {
            echo 'Build succeeded!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
