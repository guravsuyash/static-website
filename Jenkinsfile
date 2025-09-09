pipeline {
  agent any

  environment {
    REGISTRY = 'registry.shuttlewhizz.com'
    IMAGE_NAME = 'webapp-staging'
    NAMESPACE = 'webapps-staging'
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
              docker build --no-cache --pull -t ${IMAGE_TAG} .
              docker push ${IMAGE_TAG}
              docker tag ${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:latest
              docker push ${REGISTRY}/${IMAGE_NAME}:latest
            """
          }
        }
      }
    }

    stage('Create TLS Secret from Files') {
      environment {
        SECRET_NAME = 'tls-secret-staging'
      }
      steps {
        withCredentials([
          file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG'),
          file(credentialsId: 'kube-staging-ca', variable: 'TLS_CRT_FILE'),
          file(credentialsId: 'kube-staging-priv', variable: 'TLS_KEY_FILE')
        ]) {
          sh '''
            kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
            openssl x509 -in "$TLS_CRT_FILE" -noout -text >/dev/null || { echo "Invalid TLS certificate"; exit 1; }
            openssl rsa -in "$TLS_KEY_FILE" -check >/dev/null 2>&1 || { echo "Invalid TLS private key"; exit 1; }

            kubectl create secret tls "$SECRET_NAME" \
              --cert="$TLS_CRT_FILE" \
              --key="$TLS_KEY_FILE" \
              -n "$NAMESPACE" \
              --dry-run=client -o yaml | kubectl apply -f -
          '''
        }
      }
    }

    stage('Install NGINX Ingress Controller') {
      steps {
        withCredentials([
          file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')
        ]) {
          sh '''
            kubectl cluster-info || { echo "Cluster access failed"; exit 1; }
            kubectl get namespace ingress-nginx || kubectl create namespace ingress-nginx
            kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
          '''
        }
      }
    }

    stage('Create Kubernetes Registry Secret') {
      steps {
        withCredentials([
          file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG'),
          usernamePassword(credentialsId: REGISTRY_CREDENTIAL_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')
        ]) {
          sh '''
            kubectl create secret docker-registry regcred \
              --docker-server="${REGISTRY}" \
              --docker-username="${DOCKER_USER}" \
              --docker-password="${DOCKER_PASS}" \
              --docker-email="suyash@supporthives.com" \
              -n ${NAMESPACE} \
              --dry-run=client -o yaml | kubectl apply -f -
          '''
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
      script {
        withCredentials([file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
          def deploymentExists = sh(
            script: "kubectl -n ${NAMESPACE} get deployment ${IMAGE_NAME} --ignore-not-found",
            returnStatus: true
          ) == 0

          if (deploymentExists) {
            def containerName = sh(
              script: "kubectl -n ${NAMESPACE} get deployment ${IMAGE_NAME} -o jsonpath='{.spec.template.spec.containers[0].name}'",
              returnStdout: true
            ).trim()

            def prevImage = sh(
              script: "kubectl -n ${NAMESPACE} get deployment ${IMAGE_NAME} -o jsonpath='{.spec.template.spec.containers[0].image}'",
              returnStdout: true
            ).trim()

            if (containerName && prevImage) {
              sh """
                kubectl -n ${NAMESPACE} set image deployment/${IMAGE_NAME} ${containerName}=${prevImage}
                kubectl -n ${NAMESPACE} rollout status deployment/${IMAGE_NAME}
              """
            }
          }
          mail(
            to: 'suyash@supporthives.com',
            subject: "❌ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: """\
            The build for ${env.JOB_NAME} #${env.BUILD_NUMBER} has FAILED.

            Check console output at ${env.BUILD_URL}
            """.stripIndent()
          )
        }
      }
    }

    success {
      script {
        echo "Deployment successful!"
            mail(
                to: 'suyash@supporthives.com',
                subject: "✅ Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """\
                The build for ${env.JOB_NAME} #${env.BUILD_NUMBER} was successful.
                
                Check console output at ${env.BUILD_URL}
                """.stripIndent()
            )
      }
    }
    
    always {
      script {
        echo "Cleaning up temporary files..."
      }
    }
  }
}
