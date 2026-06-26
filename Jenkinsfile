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

        stage('SonarQube Analysis') {
            steps {
                // withSonarQubeEnv tells Jenkins to inject the URL and Token we configured
                // The name 'sonar-server' must match exactly what you named it in Jenkins system config
                withSonarQubeEnv('sonar-server') {
                    // We tell Maven to run the Sonar scanner. It will automatically find the compiled code.
                    sh './mvnw sonar:sonar -Dsonar.projectKey=spring-petclinic -Dsonar.projectName="Spring Petclinic"'
                }
            }
        }
    }
}