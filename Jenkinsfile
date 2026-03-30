pipeline {
    agent any
    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/ismnish/DevOps-Project-Two-Tier-Flask-Application.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }
        stage('Deploy with Docker Compose') {
            steps {
                sh 'docker compose down --remove-orphans || true'
                sh 'docker rm -f mysql two-tier-app || true'
                sh 'docker compose up -d --build'
            }
        }
    }
}
