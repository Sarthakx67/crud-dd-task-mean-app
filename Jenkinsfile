pipeline {
  agent any

  environment {
    DOCKERHUB_USER = 'sarthak6700'
    BACKEND_IMAGE  = "${DOCKERHUB_USER}/crud-dd-task-backend:latest"
    FRONTEND_IMAGE = "${DOCKERHUB_USER}/crud-dd-task-frontend:latest"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Images') {
      steps {
        sh "docker build -t ${BACKEND_IMAGE} backend"
        sh "docker build -t ${FRONTEND_IMAGE} frontend"
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_PASSWORD'
        )]) {
          sh '''
            echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_USERNAME --password-stdin
            docker push ${BACKEND_IMAGE}
            docker push ${FRONTEND_IMAGE}
          '''
        }
      }
    }

    stage('Deploy Locally') {
      steps {
        sh '''
          docker pull ${BACKEND_IMAGE}
          docker pull ${FRONTEND_IMAGE}
          docker compose down
          docker compose up -d
        '''
      }
    }

  }
}