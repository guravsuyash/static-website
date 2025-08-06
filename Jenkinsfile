pipeline {
  agent any

  environment {
    IMAGE_TAG = "${GIT_COMMIT}"
    REGISTRY = "registry.shuttlewhizz.com"
    REPO = "${REGISTRY}/myapp"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Push Image') {
      steps {
        script {
          docker.build("${REPO}:${IMAGE_TAG}")
                .push()
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        script {
          sh "kubectl set image deployment/web-deployment web=${REPO}:${IMAGE_TAG} -n webapps"
        }
      }
    }
  }

  post {
    failure {
      echo 'Deployment failed. Manual rollback required.'
    }
  }
}
