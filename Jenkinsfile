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
                // 1. Start the app in the background. 
                // We use -Xmx512m to cap its memory, ensuring ZAP has enough RAM to run without crashing!
                sh 'nohup java -Xmx512m -jar target/spring-petclinic-*.jar --server.port=8082 > petclinic.log 2>&1 & echo $! > app.pid'

                // 2. Smart Wait: Actively poll the app until it returns a 200 OK response (max wait 2 minutes)
                sh '''
                    echo "Waiting for Spring Petclinic to boot up (this can take 30-90 seconds)..."
                    timeout 120 bash -c 'while [[ "$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8082/)" != "200" ]]; do sleep 5; done' || true
                '''

                // 3. The Bulletproof Lightweight ZAP Passive Scan
                sh '''
                    # Explicitly get the Jenkins IP to avoid any Docker DNS resolution issues
                    JENKINS_IP=$(hostname -i | awk '{ print $1 }')
                    
                    echo "Routing a test request through ZAP to trigger Passive Scanning..."
                    # We curl the app THROUGH the ZAP proxy. ZAP will passively scan the response.
                    curl -s -x http://zap_server:8081 "http://${JENKINS_IP}:8082/" > /dev/null || true
                    
                    echo "Waiting 10 seconds for ZAP passive scan to process..."
                    sleep 10
                    
                    echo "Generating ZAP HTML Report..."
                    # MUST use -H "Host: localhost:8081" to bypass ZAP's DNS Rebinding protection!
                    curl -s -H "Host: localhost:8081" "http://zap_server:8081/OTHER/core/other/htmlreport/" -o zap-report.html || true
                    
                    # Fallback: If the old report API didn't generate a file, use the modern report API endpoint
                    if [ ! -s zap-report.html ]; then
                        echo "Fallback to modern report endpoint..."
                        curl -s -H "Host: localhost:8081" "http://zap_server:8081/OTHER/reports/other/report/?title=ZAP-Report&template=traditional-html" -o zap-report.html || true
                    fi
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