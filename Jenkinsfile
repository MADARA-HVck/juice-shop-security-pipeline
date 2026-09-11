pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                echo '=== Code source récupéré depuis GitHub ==='
                sh 'pwd'
                sh 'git rev-parse --short HEAD'
            }
        }

        stage('Environment Check') {
            steps {
                echo '=== Vérification de l environnement ==='
                sh 'node --version'
                sh 'npm --version'
                sh 'git --version'
                sh 'docker --version'
            }
        }
    }

    post {
        always {
            echo "=== Build #${BUILD_NUMBER} terminé : ${currentBuild.currentResult} ==="
        }
    }
}
