pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /var/run/docker.sock:/var/run/docker.sock --network hafalat-devops_hafalat-network'
        }
    }

    environment {
        IMAGE_NAME = 'soumayasayfoudine/hafalat-frontend'
        VERSION = "1.0.${BUILD_NUMBER}"
        SCANNER_HOME = tool 'SonarQubeScanner'
    }

    stages {

        stage('Install Dependencies') {
            steps {
                script {
                    sh 'npm ci'
                }
            }
        }

        stage('Lint') {
            steps {
                script {
                    sh 'npm run lint || echo "Linting completed with warnings"'
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    sh '''
                        apk add --no-cache chromium
                        export CHROME_BIN=/usr/bin/chromium-browser
                        npm run test -- --watch=false --browsers=ChromeHeadless --code-coverage
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('SonarQube Analysis') {
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
                script {
                    sh 'apk add --no-cache docker || true'
                    sh 'docker build -t $IMAGE_NAME:$VERSION .'
                    sh 'docker tag $IMAGE_NAME:$VERSION $IMAGE_NAME:latest'
                }
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
            slackSend (
                color: '#00FF00',
                message: "✅ SUCCESS: Frontend Pipeline #${BUILD_NUMBER}\nProject: ${env.JOB_NAME}\nDuration: ${currentBuild.durationString}\nBuild: ${env.BUILD_URL}"
            )
        }
        failure {
            slackSend (
                color: '#FF0000',
                message: "❌ FAILED: Frontend Pipeline #${BUILD_NUMBER}\nProject: ${env.JOB_NAME}\nDuration: ${currentBuild.durationString}\nBuild: ${env.BUILD_URL}"
            )
        }
        unstable {
            slackSend (
                color: '#FFFF00',
                message: "⚠️ UNSTABLE: Frontend Pipeline #${BUILD_NUMBER}\nProject: ${env.JOB_NAME}\nBuild: ${env.BUILD_URL}"
            )
        }
    }
}
