pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                // This step automatically checks out the code from the Git repo
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                // The Maven wrapper (mvnw) is included in the project. 
                // We use it to compile the code and run unit tests.
                // chmod +x ensures the wrapper has permission to execute.
                sh 'chmod +x mvnw'
                sh './mvnw clean package'
            }
        }
    }
}