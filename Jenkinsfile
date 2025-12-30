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
                echo "Building branch: ${BRANCH_NAME}"
            }
        }

        stage('Build & Unit Tests') {
            steps {
                configFileProvider([configFile(
                    fileId: 'maven-settings-nexus',
                    variable: 'MAVEN_SETTINGS'
                )]) {
                    sh 'mvn clean verify -s $MAVEN_SETTINGS'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube server') {
                    configFileProvider([configFile(
                        fileId: 'maven-settings-nexus',
                        variable: 'MAVEN_SETTINGS'
                    )]) {
                        sh """
                          mvn sonar:sonar \
                          -Dsonar.projectKey=springboot-app-${BRANCH_NAME} \
                          -s \$MAVEN_SETTINGS
                        """
                    }
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

        /* ================= SNAPSHOT DEPLOY ================= */

        stage('Deploy SNAPSHOT to Nexus') {
            when {
                branch 'develop'
            }
            steps {
                configFileProvider([configFile(
                    fileId: 'maven-settings-nexus',
                    variable: 'MAVEN_SETTINGS'
                )]) {
                    sh 'mvn deploy -DskipTests -s $MAVEN_SETTINGS'
                }
            }
        }

        /* ================= RELEASE DEPLOY ================= */

        stage('Deploy RELEASE to Nexus') {
            when {
                branch 'main'
            }
            steps {
                input message: "Deploy RELEASE artifact to Nexus?"

                configFileProvider([configFile(
                    fileId: 'maven-settings-nexus',
                    variable: 'MAVEN_SETTINGS'
                )]) {
                    sh 'mvn deploy -DskipTests -s $MAVEN_SETTINGS'
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline succeeded for branch: ${BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline failed for branch: ${BRANCH_NAME}"
        }
    }
}
