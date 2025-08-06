pipeline {
  agent any

  environment {
    REGISTRY = 'registry.shuttlewhizz.com'
    IMAGE_NAME = 'webapp'
    NAMESPACE = 'webapps'
    KUBECONFIG_CREDENTIAL_ID = 'kubeconfig-id'
    REGISTRY_CREDENTIAL_ID = 'Docker-credentials'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        script {
          def GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          def IMAGE_TAG = "${REGISTRY}/${IMAGE_NAME}:${GIT_SHA}"

          withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIAL_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh """
              echo \$DOCKER_PASS | docker login ${REGISTRY} -u \$DOCKER_USER --password-stdin
              docker build -t ${IMAGE_TAG} .
              docker push ${IMAGE_TAG}
              docker tag ${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:latest
              docker push ${REGISTRY}/${IMAGE_NAME}:latest
            """
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        script {
          def GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          def IMAGE_TAG = "${REGISTRY}/${IMAGE_NAME}:${GIT_SHA}"

          withCredentials([file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
            sh """
              kubectl apply -f k8s -n ${NAMESPACE}
              kubectl set image deployment/${IMAGE_NAME} ${IMAGE_NAME}=${IMAGE_TAG} -n ${NAMESPACE}
              kubectl rollout status deployment/${IMAGE_NAME} -n ${NAMESPACE}
            """
          }
        }
      }
    }
  }

  post {
    failure {
      steps {
        script {
          def GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          def IMAGE_TAG = "${REGISTRY}/${IMAGE_NAME}:${GIT_SHA}"

          withCredentials([file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
            echo "Rolling back to previous image..."
            sh """
              PREV_IMAGE=\$(kubectl -n ${NAMESPACE} get deployment ${IMAGE_NAME} -o=jsonpath='{.spec.template.spec.containers[0].image}')
              kubectl -n ${NAMESPACE} set image deployment/${IMAGE_NAME} ${IMAGE_NAME}=\$PREV_IMAGE
              kubectl -n ${NAMESPACE} rollout status deployment/${IMAGE_NAME}
            """
          }
        }
      }
    }
  }
}
