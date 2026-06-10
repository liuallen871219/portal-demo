pipeline {

    agent any

    stages {

        stage('Build') {
            steps {
                sh '''
                docker build \
                  -t 172.17.69.81:8088/portal/portal:${BUILD_NUMBER} \
                  .
                '''
            }
        }

        stage('Push') {
            steps {
                sh '''
                docker push \
                  172.17.69.81:8088/portal/portal:${BUILD_NUMBER}
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                kubectl set image deployment/portal \
                  portal=172.17.69.81:8088/portal/portal:${BUILD_NUMBER}

                kubectl rollout status deployment/portal
                '''
            }
        }
    }
}
