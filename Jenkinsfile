pipeline {
    agent any

    tools {
        // We need to tell Jenkins to use Java 17, which Spring Petclinic requires.
        // We will configure this tool name ('jdk17') in Jenkins shortly.
        jdk 'jdk17' 
    }

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