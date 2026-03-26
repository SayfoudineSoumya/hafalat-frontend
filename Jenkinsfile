pipeline {
    agent {
        docker {
            image 'cypress/browsers:node-20.11.0-chrome-121.0.6167.85-1-ff-120.0.1-edge-121.0.2277.83-1'
            args '-v /var/run/docker.sock:/var/run/docker.sock --network hafalat-devops_hafalat-network'
        }
    }

    environment {
        IMAGE_NAME = 'soumayasayfoudine/hafalat-frontend'
        VERSION = "1.0.${BUILD_NUMBER}"
    }

    stages {
        // stage('Checkout') {
        //     steps {
        //         checkout scm
        //     }
        // }
        stage('Install Dependencies') {
            steps {
                script {
                    sh 'npm install'
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    sh 'npm run test -- --watch=false --browsers=ChromeHeadless --code-coverage'
                }
            }
        }

        stage('Build Angular App') {
            steps {
                sh 'npm run build -- --configuration production'
            }
        }

        stage('SonarQube Analysis') {
            agent any
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
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

        stage('Docker Build') {
            agent any
            steps {
                sh 'docker build -t $IMAGE_NAME:$VERSION .'
                sh 'docker tag $IMAGE_NAME:$VERSION $IMAGE_NAME:latest'
            }
        }

        stage('Docker Push') {
            agent any
            steps {
                withCredentials([usernamePassword(credentialsId: 'DockerHubjenkinsCI', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                    sh 'docker push $IMAGE_NAME:$VERSION'
                    sh 'docker push $IMAGE_NAME:latest'
                }
            }
        }

        stage('Deploy') {
            agent any
            steps {
                sh '''
                    docker pull $IMAGE_NAME:latest
                    docker stop hafalat-frontend || true
                    docker rm hafalat-frontend || true
                    docker run -d \
                      --name hafalat-frontend \
                      --network hafalat-devops_hafalat-network \
                      -p 4200:80 \
                      $IMAGE_NAME:latest
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Frontend Build Success'
        }
        failure {
            echo '❌ Frontend Build Failed'
        }
        unstable {
            echo '⚠️ Frontend Build Unstable'
        }
    }
}
