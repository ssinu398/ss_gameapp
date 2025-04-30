pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
    }
    
    stages {
        stage('GIT CHECKOUT') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Gokul-1701/Gaming-Application.git'
            }
        }
        
         stage('MAVEN COMPILE') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('MAVEN TEST') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('TRIVY FILESYSTEM SCAN') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                sh '''
                $SCANNER_HOME/bin/sonar-scanner \
                -Dsonar.projectKey=Gaming \
                -Dsonar.projectName=Gaming \
                -Dsonar.java.binaries=.
                '''
                }
            }
        }
        
        stage('QUALITY GATE') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
        
        stage('MAVEN BUILD') {
            steps {
               sh "mvn package"
            }
        }
        
        stage('BUILD & TAG DOCKER IMAGE') {
            steps {
               script {
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t gokulr17/gaming:latest ."
                    }
               }
            }
        }
        
        stage('IMAGE SCANNING') {
            steps {
                sh "trivy image --scanners vuln --format table -o trivy-image-report.html gokulr17/gaming:latest"
            }
            
        }
        
        stage('DOCKER DEPLOYMENT') {
            steps {
                sh "docker run -d -p 8091:8080 gokulr17/gaming:latest"
            }
        }
    }
}
