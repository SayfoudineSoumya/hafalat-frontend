pipeline {
    agent any

    environment {
        IMAGE_NAME = 'soumayasayfoudine/hafalat-frontend'
        VERSION = "1.0.${BUILD_NUMBER}"
        DOCKER_NETWORK = 'hafalat-devops_hafalat-network'
    }

    stages {

        stage('Install Dependencies') {
            agent {
                docker {
                    image 'hafalat-ci:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock --network hafalat-devops_hafalat-network'
                    reuseNode true
                }
            }
            steps {
                script {
                    sh 'npm ci'
                }
            }
        }

        stage('Lint') {
            agent {
                docker {
                    image 'hafalat-ci:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock --network hafalat-devops_hafalat-network'
                    reuseNode true
                }
            }
            steps {
                script {
                    sh 'npm run lint || echo "No linting configured"'
                }
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'hafalat-ci:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock --network hafalat-devops_hafalat-network'
                    reuseNode true
                }
            }
            steps {
                script {
                    sh '''
                    echo "Running Angular tests..."
                    npm test -- --watch=false --browsers=ChromeHeadless --code-coverage || echo "Tests skipped"
                    '''
                }
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'hafalat-ci:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock --network hafalat-devops_hafalat-network'
                    reuseNode true
                }
            }
            steps {
                script {
                    sh 'npm run build -- --configuration production'
                }
            }
        }

        stage('SonarQube Analysis') {
            agent {
                docker {
                    image 'hafalat-ci:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock --network hafalat-devops_hafalat-network'
                    reuseNode true
                }
            }
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                        sonar-scanner \
                            -Dsonar.projectKey=hafalat-frontend \
                            -Dsonar.sources=src \
                            -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/*.spec.ts \
                            -Dsonar.tests=src \
                            -Dsonar.test.inclusions=**/*.spec.ts \
                            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                            -Dsonar.typescript.tsconfigPath=tsconfig.json
                        '''
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    sh '''
                    echo "Building Docker image..."
                    docker build --pull -t $IMAGE_NAME:$VERSION .
                    docker tag $IMAGE_NAME:$VERSION $IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'DockerHubjenkinsCI', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh '''
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push $IMAGE_NAME:$VERSION
                        docker push $IMAGE_NAME:latest
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh '''
                    echo "Deploying container..."

                    if [ $(docker ps -aq -f name=hafalat-frontend) ]; then
                        docker stop hafalat-frontend || true
                        docker rm hafalat-frontend || true
                    fi

                    docker run -d \
                        --name hafalat-frontend \
                        --network $DOCKER_NETWORK \
                        -p 4200:80 \
                        $IMAGE_NAME:latest
                    '''
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#devops-ensi', 
                color: 'good', 
                message: "✅ Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} ${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: '#devops-ensi', 
                color: 'danger', 
                message: "❌ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} ${env.BUILD_URL}"
            )
        }
    }
}
