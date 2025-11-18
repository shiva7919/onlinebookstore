pipeline {
    agent any

    environment {
        SONARQUBE_URL = 'http://54.209.26.217:9000'
        NEXUS_URL     = 'http://54.209.26.217:8081'
        DOCKER_IMAGE  = "shivasarla2398/onlinebookstore"
        VERSION       = "${env.BUILD_NUMBER}"
        SONAR_WAIT_ATTEMPTS = "12"
        SONAR_WAIT_SLEEP    = "5"
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

        stage('Wait for Sonar') {
            steps {
                echo "Waiting for SonarQube at $SONARQUBE_URL ..."
                sh '''
                    attempts=${SONAR_WAIT_ATTEMPTS}
                    sleep_seconds=${SONAR_WAIT_SLEEP}
                    success=0
                    i=0
                    while [ $i -lt $attempts ]; do
                      i=$((i+1))
                      echo "Attempt $i/$attempts: checking $SONARQUBE_URL ..."
                      if curl --silent --head --fail --connect-timeout 5 $SONARQUBE_URL >/dev/null 2>&1; then
                        echo "Sonar reachable."
                        success=1
                        break
                      else
                        echo "Sonar not reachable yet. Sleeping ${sleep_seconds}s..."
                        sleep ${sleep_seconds}
                      fi
                    done

                    if [ "$success" -ne 1 ]; then
                      echo "ERROR: SonarQube did not become reachable at $SONARQUBE_URL"
                      exit 1
                    fi
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Running SonarQube Analysis..."
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=onlinebookstore \
                          -Dsonar.host.url=$SONARQUBE_URL \
                          -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus', passwordVariable: 'NEXUS_PASS', usernameVariable: 'NEXUS_USER')]) {
                    sh '''
                        if [ -z "$NEXUS_USER" ] || [ -z "$NEXUS_PASS" ]; then
                          echo "ERROR: Nexus credentials missing"
                          exit 1
                        fi

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
                echo "Pushing Docker image..."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-user', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                        if [ -z "$DH_USER" ] || [ -z "$DH_PASS" ]; then
                          echo "ERROR: DockerHub credentials missing"
                          exit 1
                        fi

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
