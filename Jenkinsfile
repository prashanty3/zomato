pipeline {
    agent any
    tools {
        jdk 'jdk21'
        nodejs 'node23'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage ("clean workspace") {
            steps {
                cleanWs()
            }
        }
        stage ("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/prashanty3/zomato.git'
            }
        }
        stage("Sonarqube Analysis"){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage("Code Quality Gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                // First update the vulnerability database (recommended)
                dependencyCheck additionalArguments: '--updateonly --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                
                // Then perform the actual scan
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --format XML --out reports/', odcInstallation: 'DP-Check'
                
                // Publish the results
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage ("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }
        stage ("Build Docker Image") {
            steps {
                script {
                    // 1. Stop and remove any running container using the old image
                    sh '''
                    if docker ps -a --format "{{.Names}}" | grep -q "zomato"; then
                        docker stop zomato || true
                        docker rm zomato || true
                    fi
                    '''
                    
                    // 2. Remove the old image if it exists
                    sh '''
                    if docker images --format "{{.Repository}}:{{.Tag}}" | grep -q "sonalisinhawipro/zomato:latest"; then
                        docker rmi sonalisinhawipro/zomato:latest || true
                    fi
                    '''
                    
                    // 3. Build the new image
                    sh "docker build -t sonalisinhawipro/zomato:latest ."
                }
            }
        }
        stage ("Tag & Push to DockerHub") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker tag zomato sonalisinhawipro/zomato:latest "
                        sh "docker push sonalisinhawipro/zomato:latest "
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview sonalisinhawipro/zomato:latest'
                       sh 'docker-scout cves sonalisinhawipro/zomato:latest'
                       sh 'docker-scout recommendations sonalisinhawipro/zomato:latest'
                   }
                }
            }
        }
        stage ("Deploy to Container") {
            steps {
                sh 'docker run -d --name zomato -p 3000:3000 sonalisinhawipro/zomato:latest'
            }
        }
    }
    post {
    always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: """
                <html>
                <body>
                    <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                    </div>
                    <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                    </div>
                    <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                    </div>
                </body>
                </html>
            """,
            to: 'sonalisinhawipro@gmail.com',
            mimeType: 'text/html',
            attachmentsPattern: 'trivy.txt'
        }
    }
}
