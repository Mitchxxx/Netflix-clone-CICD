pipeline {
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME= tool 'SonarQubeScanner'
    }
    stages{
        stage('Clean Workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout frrom Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Mitchxxx/Netflix-clone-CICD.git'
            }
        }
        stage('Sonarqube Analysis'){
            steps{
                withSonarQubeEnv('SonarQubeSever') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage('Quality Gate'){
            steps {
                script {
                    waitForQualityGate abortPieline: false, credentialsId: 'SonarQubeToken'
                }
            }
        }
        stage('Install Dependencies'){
            steps{
                sh "npm install"
            }
        }
        stage('OWASP FS Scan'){
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan'){
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('Docker Build and Push'){
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker', toolName: 'docker'){
                        sh "docker build --build-arg TMDB_V3_API_KEY=733de66f2f8ebf885ea8575cdbf3d8cd -t netflix-clone-cicd ."
                        sh "docker tag netflix-clone-cicd mitchxxx/netflix-clone-cicd:latest "
                        sh "docker push mitchxxx/netflix-clone-cicd:latest "
                    }
                }
            }
        }
        stage('Trivy Image Scan'){
            steps {
                sh "trivy image mitchxxx/netflix-clone-cicd:latest > trivyimage.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix-clone-cicd -p 8081:80 mitchxxx/netflix-clone-cicd:latest'
            }
        }
        stage('Deploy to Kubernetes'){
            steps{
                script{
                    dir('Kubernetes') {
                       withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: ''){
                            sh "kubectl apply -f deployment.yml"
                            sh "kubectl apply -f service.yml"
                        }
                    }
                }
            }
        }

    }
    post{
        always {
            emailext attachLog: true,
                subject: "${currentBuild.result}",
                body: "Project: ${env.JOB_NAME}<br/>" +
                    "Build Number: ${env.BUILD_NUMBER}<br/>" +
                    "URL: ${env.BUILD_URL}<br/>",
                to: 'megboko@gmail.com',
                attachmentsPattern: 'trivyfs.txt, trivyimage.txt'
        }
    }
}
