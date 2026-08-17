pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node18'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'ursulan1/netflix'
        DOCKER_TAG = 'latest'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/UrsulaN1/netflix-clone-DevSecOps.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                        npm install
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=Netflix \
                            -Dsonar.projectKey=Netflix
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate(
                        abortPipeline: false,
                        credentialsId: 'Sonar-token'
                    )
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --no-progress \
                        . | tee trivyfs.txt
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x

                        printf '%s' "$DOCKER_TOKEN" | \
                        docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'tmdb-api-key',
                        variable: 'TMDB_KEY'
                    )
                ]) {
                    sh '''
                        docker build \
                            --build-arg TMDB_V3_API_KEY="$TMDB_KEY" \
                            -t "$DOCKER_IMAGE:$DOCKER_TAG" .
                    '''
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --no-progress \
                        "$DOCKER_IMAGE:$DOCKER_TAG" \
                        | tee trivyimage.txt
                '''
            }
        }

        stage('Docker Push') {
            steps {
                sh '''
                    docker push "$DOCKER_IMAGE:$DOCKER_TAG"
                '''
            }
        }

        stage('Deploy to Docker Container') {
            steps {
                sh '''
                    docker rm -f netflix || true

                    docker run \
                        -d \
                        --name netflix \
                        --restart unless-stopped \
                        -p 8081:80 \
                        "$DOCKER_IMAGE:$DOCKER_TAG"
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                dir('Kubernetes') {
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: '',
                        contextName: '',
                        credentialsId: 'k8s',
                        namespace: '',
                        restrictKubeConfigAccess: false,
                        serverUrl: ''
                    ) {
                        sh '''
                            kubectl apply -f deployment.yml
                            kubectl apply -f service.yml

                            kubectl get deployments
                            kubectl get pods
                            kubectl get services
                        '''
                    }
                }
            }
        }
    }

    post {

        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage above.'
        }

        always {
            sh 'docker logout || true'

            emailext(
                attachLog: true,
                subject: "${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    Project: ${env.JOB_NAME}<br/>
                    Build Number: ${env.BUILD_NUMBER}<br/>
                    Status: ${currentBuild.currentResult}<br/>
                    URL: ${env.BUILD_URL}<br/>
                """,
                to: 'myexample@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
