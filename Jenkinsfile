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

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
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
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                -v trivy-cache:/root/.cache/trivy \
                aquasec/trivy:latest image \
                --timeout 30m \
                --severity HIGH,CRITICAL \
                --exit-code 1 \
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