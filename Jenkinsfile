pipeline {
    agent any

    tools {
        jdk 'JDK21'
        maven 'Maven-3.9.12'
    }

    environment {
        NEXUS_CREDENTIALS_ID = 'nexus-jenkins-creds'
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Unit Tests') {
            steps {
                configFileProvider([configFile(fileId: 'maven-settings-nexus', variable: 'MAVEN_SETTINGS')]) {
                    sh 'mvn clean verify -s $MAVEN_SETTINGS'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube server') {
                    configFileProvider([configFile(fileId: 'maven-settings-nexus', variable: 'MAVEN_SETTINGS')]) {
                        sh '''
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                            -Dsonar.projectKey=springboot-app \
                            -s $MAVEN_SETTINGS
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy to Nexus (Release)') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${NEXUS_CREDENTIALS_ID}",
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {
                    configFileProvider([configFile(fileId: 'maven-settings-nexus', variable: 'MAVEN_SETTINGS')]) {
                        sh 'mvn deploy -s $MAVEN_SETTINGS'
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully: Build, SonarQube, Nexus deployment'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
