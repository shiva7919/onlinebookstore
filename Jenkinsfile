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
                    // build locally for faster reuse; will rebuild in push stage for safety
                    def tmpImage = docker.build("${DOCKER_IMAGE}:${VERSION}")
                    // remove tmpImage reference to avoid keeping it in pipeline memory
                    sh "docker rmi ${DOCKER_IMAGE}:${VERSION} || true"
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo "Pushing Docker image..."
                // Use the Jenkins credential with ID 'dockerhub'
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    script {
                        // Basic sanity check: ensure the credential user matches the image namespace to avoid denied pushes
                        def expectedNamespace = env.DOCKER_IMAGE.split('/')[0]
                        echo "Image namespace expected: ${expectedNamespace}"
                        echo "Credential username: ${DH_USER}"

                        if (DH_USER != expectedNamespace) {
                            error("""
Credential username '${DH_USER}' does not match the image namespace '${expectedNamespace}'.
Either:
  • Use a credential whose username is '${expectedNamespace}', OR
  • Change DOCKER_IMAGE to use the username in the credential, OR
  • Add the credential user as a collaborator on the '${expectedNamespace}/onlinebookstore' repo on Docker Hub.
""")
                        }

                        try {
                            // rebuild here to get a stage-scoped image object and then push
                            def img = docker.build("${DOCKER_IMAGE}:${VERSION}")
                            docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                                img.push("${VERSION}")
                                img.push("latest")
                            }
                            echo "Docker image pushed successfully: ${DOCKER_IMAGE}:${VERSION} and :latest"
                        } catch (err) {
                            echo "ERROR: Docker push failed. Possible causes: invalid credentials, wrong credential ID, user lacks push permission, or repo does not exist on Docker Hub."
                            throw err
                        } finally {
                            sh "docker rmi ${DOCKER_IMAGE}:${VERSION} || true"
                            sh "docker rmi ${DOCKER_IMAGE}:latest || true"
                        }
                    }
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
