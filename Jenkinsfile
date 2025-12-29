pipeline {
    agent any

    tools {
        maven 'Maven-3.9.12'
        jdk 'JDK21'
    }

    environment {
        // Jenkins credentials ID for Nexus (username/password)
        NEXUS_CREDENTIALS = credentials('nexus-jenkins-creds')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Unit Tests') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube server') {
                    sh '''
                      mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                      -Dsonar.projectKey=springboot-app
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy to Nexus (Release)') {
            steps {
                // Use injected Jenkins credentials for Nexus deployment
                sh '''
                  mvn deploy \
                  -Dnexus.username=${NEXUS_CREDENTIALS_USR} \
                  -Dnexus.password=${NEXUS_CREDENTIALS_PSW}
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
