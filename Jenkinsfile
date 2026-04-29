pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t mi-app:latest .'
            }
        }

        stage('Static Analysis') {
            steps {
                echo 'Aquí irá SonarQube'
            }
        }

        stage('Security Scan') {
            steps {
                echo 'Aquí irá Trivy'
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                docker stop mi-app || true
                docker rm mi-app || true
                docker run -d --name mi-app -p 80:80 mi-app:latest
                '''
            }
        }
    }

    post {
        always {
            echo 'Limpiando entorno...'
            cleanWs()
        }

        failure {
            echo 'Falló el pipeline.'
        }
    }
}