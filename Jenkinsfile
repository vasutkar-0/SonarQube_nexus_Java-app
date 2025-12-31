pipeline {
    agent any

    tools {
        jdk 'JDK21'
        maven 'Maven-3.9.12'
    }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-21-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        IMAGE_NAME = ""  // will be set dynamically
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Building branch: ${env.BRANCH_NAME}"
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
                          -Dsonar.projectKey=springboot-app-${env.BRANCH_NAME} \
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

        // Download artifact from Nexus for Docker image
        stage('Download Artifact from Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-jenkins-creds',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    script {
                        // Change version if needed or parameterize
                        def artifactVersion = '4.0.0'
                        def artifactUrl = "http://172.25.3.113:8081/repository/maven-releases/com/example/demo/${artifactVersion}/demo-${artifactVersion}.jar"
                        echo "Downloading artifact from Nexus: ${artifactUrl}"

                        sh """
                        curl -u $NEXUS_USER:$NEXUS_PASS -L -o demo-app.jar ${artifactUrl}
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_NAME = "mydockerhubuser/myapp:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    echo "Building Docker image: ${env.IMAGE_NAME}"
                    sh "docker build -t ${env.IMAGE_NAME} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }
                    sh "docker push ${env.IMAGE_NAME}"
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                script {
                    // Replace IMAGE_TAG placeholder in docker-compose.yml with actual image tag
                    sh """
                    sed 's#REPLACE_IMAGE_TAG#${env.IMAGE_NAME}#g' docker-compose.yml > docker-compose-run.yml
                    docker-compose -f docker-compose-run.yml down || true
                    docker-compose -f docker-compose-run.yml up -d
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline succeeded for branch: ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline failed for branch: ${env.BRANCH_NAME}"
        }
    }
}
