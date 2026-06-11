pipeline {

agent any

options {
    buildDiscarder(
        logRotator(
            numToKeepStr: '20'
        )
    )
}

stages {

    stage('Build') {
        steps {
            sh '''
            docker build \
              -t 172.17.69.81:8088/portal/portal:${BUILD_NUMBER} .
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
            helm upgrade --install portal helm/portal \
              --set image.tag=${BUILD_NUMBER}

            kubectl rollout status deployment/portal
            '''
        }
    }

    stage('Verify') {
        steps {
            sh '''
            kubectl get pods -o wide
            kubectl get svc
            '''
        }
    }
}

}

