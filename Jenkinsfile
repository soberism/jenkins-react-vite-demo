pipeline {
    agent any

    tools {
        nodejs 'node22'
    }

    environment {
        REGISTRY = 'localhost:8089'
        HARBOR_PROJECT = 'demo'
        IMAGE_NAME = 'jenkins-react-vite-demo'
        IMAGE_REPO = 'localhost:8089/demo/jenkins-react-vite-demo'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh 'node -v'
                sh 'npm -v'
                sh 'npm ci'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker version
                    docker build -t ${IMAGE_REPO}:${BUILD_NUMBER} .
                    docker tag ${IMAGE_REPO}:${BUILD_NUMBER} ${IMAGE_REPO}:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-admin', usernameVariable: 'HARBOR_USERNAME', passwordVariable: 'HARBOR_PASSWORD')]) {
                    sh '''
                        echo "${HARBOR_PASSWORD}" | docker login ${REGISTRY} -u "${HARBOR_USERNAME}" --password-stdin
                        docker push ${IMAGE_REPO}:${BUILD_NUMBER}
                        docker push ${IMAGE_REPO}:latest
                    '''
                }
            }
        }

        stage('Deploy to k8s') {
            steps {
                sh '''
                    kubectl apply -f k8s/
                    kubectl set image deployment/${IMAGE_NAME} web=${IMAGE_REPO}:${BUILD_NUMBER}
                    kubectl rollout status deployment/${IMAGE_NAME}
                '''
            }
        }
    }

    post {
        success {
            echo 'React + Vite image pushed to Harbor and deployed to k8s.'
            echo 'Open: http://localhost'
        }
        failure {
            echo 'Build failed. Check Console Output for details.'
        }
    }
}
