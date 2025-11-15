pipeline {
    agent any

    environment {
        SONARQUBE_URL = 'http://sonar:9000'
        NEXUS_URL     = 'http://nexus:8081'
        DOCKER_IMAGE  = "kishangollamudi/onlinebookstore"
        VERSION       = "${env.BUILD_NUMBER}"
    }

    tools {
        maven 'Maven-3'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                echo "Running Maven Build..."
                sh 'mvn clean package'
            }
            post {
                always {
                    echo "Publishing test results (allow empty)"
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube Analysis..."
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=onlinebookstore \
                        -Dsonar.host.url=${SONARQUBE_URL} \
                        -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                echo "Uploading WAR to Nexus..."
                withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                        mvn deploy \
                        -DskipTests \
                        -Dnexus.user=${NEXUS_USER} \
                        -Dnexus.pass=${NEXUS_PASS} \
                        -s settings.xml
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                script {
                    docker.build("${DOCKER_IMAGE}:${VERSION}")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo "Pushing Docker image to DockerHub..."
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo ${DH_PASS} | docker login -u ${DH_USER} --password-stdin
                        docker push ${DOCKER_IMAGE}:${VERSION}
                        docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

    }

    post {
        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
    }
}
