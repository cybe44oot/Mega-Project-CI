pipeline {
    agent any
    
    tools {
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/cybe44oot/Mega-Project-CI.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Testing') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('Trivy FS Scan') {
            when {
                expression { false }
            }
            
            steps {
                sh "trivy fs --format table -o fs-report.html ."
            }
        }
        
        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=gcbank \
                        -Dsonar.projectName=gcbank \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'alhobab', maven: 'maven3', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        
        stage('Docker Image Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub') {
                        sh "docker build -t alhobab/bankapp:$IMAGE_TAG ."
                    }
                }
            }
        }
        
        stage('Scan Image') {
            when { 
                expression { false}
            }
            steps {
                sh "trivy image --format table -o image-report.html alhobab/bankapp:$IMAGE_TAG"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub') {
                        sh "docker push alhobab/bankapp:$IMAGE_TAG"
                    }
                }
            }
        }
        
        stage('Update Manifest File in Mega-Project-CD') {
            steps {
                script {
                    // Clean workspace before starting
                    cleanWs()

                    withCredentials([usernamePassword(credentialsId: 'git', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh '''
                            # Clone the Mega-Project-CD repository
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/cybe44oot/Mega-Project-CD.git
                            
                            # Update the image tag in the manifest.yaml file
                            cd Mega-Project-CD
                            sed -i "s|alhobab/bankapp:.*|alhobab/bankapp:${IMAGE_TAG}|" Manifest/manifest.yaml
                            
                            # Confirm changes
                            echo "Updated manifest file contents:"
                            cat Manifest/manifest.yaml
                            
                            # Commit and push the changes
                            git config user.name "cybe44oot"
                            git config user.email "lswagy22@gmail.com"
                            git add Manifest/manifest.yaml
                            git commit -m "Update image tag to ${IMAGE_TAG}"
                            git push origin main
                        '''
                    }
                }
            }
        }
    }
}
