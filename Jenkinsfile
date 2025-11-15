pipeline {
    agent any

    environment {
        // Tools
        MAVEN_HOME   = tool "Maven-3"
        SONAR_SCANNER = "SonarScanner"

        // SonarQube Server config (using Jenkins integration)
        SONARQUBE = "My-Sonar"

        // Docker Hub Credential
        DOCKER_CRED = credentials('dockerhub-user')

        // Nexus Credentials
        NEXUS_USER = credentials('nexus-user')
        NEXUS_PASS = credentials('nexus-pass')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: 'https://github.com/KishanGollamudi/onlinebookstore.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('My-Sonar') {
                    sh """
                       ${MAVEN_HOME}/bin/mvn clean verify sonar:sonar \
                       -Dsonar.projectKey=onlinebookstore \
                       -Dsonar.host.url=$SONAR_HOST_URL
                    """
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

        stage('Build with Maven') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn clean package"
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                sh """
                ${MAVEN_HOME}/bin/mvn deploy \
                  -DaltDeploymentRepository=nexus::default::http://nexus:8081/repository/maven-releases/ \
                  -Dusername=$NEXUS_USER -Dpassword=$NEXUS_PASS
                """
            }
        }

        stage('Build Docker Image for Tomcat Deployment') {
            steps {
                sh """
                docker build -t ${DOCKER_CRED_USR}/onlinebookstore:latest .
                """
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                sh """
                    echo $DOCKER_CRED_PSW | docker login -u $DOCKER_CRED_USR --password-stdin
                    docker push ${DOCKER_CRED_USR}/onlinebookstore:latest
                """
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check logs."
        }
    }
}
