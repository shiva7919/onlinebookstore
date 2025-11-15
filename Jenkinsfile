pipeline {
    agent any

    tools {
        maven 'Maven-3'
    }

    environment {
        SONARQUBE_ENV = 'My-Sonar'
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        skipDefaultCheckout(true)
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${env.SONARQUBE_ENV}") {
                    sh """
                        mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=onlinebookstore \
                        -Dsonar.host.url=http://sonar:9000
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build WAR with Maven') {
            steps {
                sh "mvn clean package"
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                        curl -v -u $NEXUS_USER:$NEXUS_PASS --upload-file target/onlinebookstore.war \
                        http://nexus:8081/repository/maven-releases/onlinebookstore/onlinebookstore.war
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t onlinebookstore:v1 ."
                }
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo $DH_PASS | docker login -u $DH_USER --password-stdin
                        docker tag onlinebookstore:v1 $DH_USER/onlinebookstore:v1
                        docker push $DH_USER/onlinebookstore:v1
                    """
                }
            }
        }
    }
}
