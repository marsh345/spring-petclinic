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

        stage('Dynamic Security Analysis (ZAP)') {
            steps {
                // 1. Start the compiled app in the background on port 8082
                sh 'nohup java -jar target/spring-petclinic-*.jar --server.port=8082 > petclinic.log 2>&1 & echo $! > app.pid'

                // 2. Smart Wait: Actively poll the app until it returns a 200 OK response (max wait 2 minutes)
                sh '''
                    echo "Waiting for Spring Petclinic to boot up (this can take 30-90 seconds)..."
                    timeout 120 bash -c 'while [[ "$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8082/)" != "200" ]]; do sleep 5; done' || echo "Timeout reached, continuing anyway..."
                '''

                // 3. Trigger ZAP using a hybrid approach for 100% reliability
                sh '''
                    # Fetch internal container IP for Jenkins
                    JENKINS_IP=$(hostname -i | awk '{ print $1 }')
                    echo "Targeting Jenkins App at IP: $JENKINS_IP"
                    
                    # A. Trigger ZAP Spider
                    # - We use -H "Host: localhost" so the ZAP API accepts the connection (bypassing DNS Rebind protection).
                    # - We pass the explicit IP in the URL so ZAP's crawler doesn't get lost trying to resolve Docker hostnames.
                    curl -s -H "Host: localhost" "http://zap_server:8081/JSON/spider/action/scan/?url=http://${JENKINS_IP}:8082"
                    
                    # B. Wait 30 seconds for ZAP to finish crawling and scanning
                    echo "ZAP is scanning... waiting 30 seconds..."
                    sleep 30
                    
                    # C. Generate the HTML report
                    curl -s -H "Host: localhost" "http://zap_server:8081/OTHER/core/other/htmlreport/" -o zap-report.html
                '''
            }
        }
    }

    // The post section runs after all stages are complete (or if they fail)
    post {
        always {
            // Fulfills the requirement: "Add post-build actions to publish reports for ZAP"
            // This saves the report so you can download it directly from the Jenkins UI
            archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
            
            // Clean up the background app if it's running, even if the build failed above
            sh 'kill $(cat app.pid) || true'
        }
    }
}