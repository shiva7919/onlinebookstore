pipeline {
    agent any

    environment {
        SONARQUBE_URL = 'http://sonar:9000'
        NEXUS_URL     = 'http://nexus:8081'
        DOCKER_IMAGE  = "shivasarla2398/onlinebookstore"
        VERSION       = "${env.BUILD_NUMBER}"
        SONAR_TOKEN   = "sqa_0c955ce40ff0005b242b802aacfb313b589e279b"
    }

    tools {
        maven 'Maven'
        // jdk 'JDK17'
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
                echo "Building with Maven..."
                sh 'mvn clean package'
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube..."
                sh '''
                    mvn sonar:sonar \
                      -Dsonar.projectKey=onlinebookstore \
                      -Dsonar.host.url=$SONARQUBE_URL \
                      -Dsonar.login=$SONAR_TOKEN
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus', passwordVariable: 'NEXUS_PASS', usernameVariable: 'NEXUS_USER')]) {

                    sh '''
                        cat <<EOF > settings.xml
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
  </servers>
</settings>
EOF
                    '''

                    sh 'mvn deploy -DskipTests -s settings.xml'
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
                echo "Pushing image to DockerHub..."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-user', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${VERSION}
                        docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    '''
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
