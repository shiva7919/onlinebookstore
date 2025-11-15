pipeline {
    agent any

    environment {
        SONAR_HOST_URL = "http://sonar:9000"
        SONAR_SCANNER = "SonarScanner"
        MAVEN_HOME = tool "Maven-3"
        DOCKERHUB_USER = credentials('dockerhub-user')
        DOCKERHUB_PASS = credentials('dockerhub-pass')
        NEXUS_USER = credentials('nexus-user')
        NEXUS_PASS = credentials('nexus-pass')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'YOUR_GITHUB_REPO_URL'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('My-Sonar') {
                    sh """
                       ${MAVEN_HOME}/bin/mvn clean verify sonar:sonar \
                       -Dsonar.projectKey=my-app \
                       -Dsonar.host.url=$SONAR_HOST_URL \
                       -Dsonar.login=$SONARQUBE_AUTH_TOKEN
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

        stage('Upload to Nexus') {
            steps {
                sh """
                ${MAVEN_HOME}/bin/mvn deploy \
                -DaltDeploymentRepository=nexus::default::http://nexus:8081/repository/maven-releases/ \
                -Dusername=$NEXUS_USER -Dpassword=$NEXUS_PASS
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKERHUB_USER}/myapp:latest .
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh """
                    echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin
                    docker push ${DOCKERHUB_USER}/myapp:latest
                """
            }
        }
    }
}
