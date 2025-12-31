pipeline {
    agent any

    tools {
        jdk 'JDK21'
        maven 'Maven-3.9.12'
    }

    environment {
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
                          mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
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

        /* ================= RELEASE DEPLOY ================= */
        stage('Deploy RELEASE to Nexus') {
            when {
                branch 'main'
            }
            steps {
                input message: "Deploy RELEASE artifact to Nexus?"

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-jenkins-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {
                    configFileProvider([configFile(
                        fileId: 'maven-settings-nexus',
                        variable: 'MAVEN_SETTINGS'
                    )]) {
                        sh 'mvn deploy -DskipTests -s $MAVEN_SETTINGS'
                    }
                }
            }
        }

        /* ================= DEPLOY MAIN FROM NEXUS ================= */
        stage('Deploy MAIN from Nexus') {
            when {
                branch 'main'
            }
            steps {
                input message: "Deploy MAIN artifact from Nexus to Jenkins server?"

                withCredentials([usernamePassword(
                    credentialsId: 'nexus-jenkins-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    script {
                        // Replace with your actual Nexus release artifact URL
                        def ARTIFACT_URL = "http://172.25.3.113:8081/repository/maven-releases/com/example/demo/4.0.0/demo-4.0.0.jar"

                        echo "Downloading artifact from Nexus: ${ARTIFACT_URL}"

                        sh """
                        curl -u $NEXUS_USER:$NEXUS_PASS -o app.jar -L ${ARTIFACT_URL}

                        echo "Stopping existing application (if any)"
                        pkill -f '.jar' || true

                        echo "Starting application on Jenkins server"
                        nohup java -jar app.jar > app.log 2>&1 &
                        """
                    }
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
