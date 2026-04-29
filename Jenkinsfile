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
                sh 'mvn test -DskipTests'
            }
        }

        stage('Static Analysis (SonarQube)') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    mvn sonar:sonar \
                    -Dsonar.projectKey=cicd-demo \
                    -Dsonar.projectName=cicd-demo
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t mi-app:latest .'
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                trivy image \
                --scanners vuln \
                --no-progress \
                --timeout 10m \
                --severity CRITICAL \
                --exit-code 0 \
                mi-app:latest
                '''
            }
        }

        stage('Deploy') {
            when {
                branch 'master'
            }
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
            cleanWs()
        }

        failure {
            echo 'Falló el pipeline.'
        }
    }
}