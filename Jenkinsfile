pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hello World'
            }
        }
        
        stage('Environment Check') {
            steps {
                sh 'java -version'
                sh 'echo "working dir: "'
                sh 'pwd'
            }
        }
    }
}