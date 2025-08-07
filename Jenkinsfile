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

    stages {
        stage('Create TLS Secret') {
            steps {
                withCredentials([
                    string(credentialsId: 'kube-worker-tls-crt', variable: 'TLS_CRT'),
                    string(credentialsId: 'kube-worker-tls-key', variable: 'TLS_KEY')
                ]) {
                    sh '''
                        echo "$TLS_CRT" > tls.crt
                        echo "$TLS_KEY" > tls.key

                        kubectl create secret tls tls-secret \
                            --cert=tls.crt \
                            --key=tls.key \
                            -n $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

                        rm tls.crt tls.key
                    '''
                }
            }
        }
    }

    stage('Install NGINX Ingress Controller') {
        steps {
            sh '''
                # Create namespace if it doesn't exist
                kubectl get namespace ingress-nginx || kubectl create namespace ingress-nginx

                # Apply the ingress-nginx manifest
                kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
            '''
        }
    }

    stage('Deploy to Kubernetes') {
      steps {
        script {
          def GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          def IMAGE_TAG = "${REGISTRY}/${IMAGE_NAME}:${GIT_SHA}"

          withCredentials([file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
            sh """
              kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
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
