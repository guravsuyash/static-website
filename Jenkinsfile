pipeline {
  agent any

  environment {
    // Docker registry URL
    REGISTRY = 'registry.shuttlewhizz.com'
    // Docker image name
    IMAGE_NAME = 'webapp'
    // Kubernetes namespace
    NAMESPACE = 'webapps'
    // Jenkins credentials IDs
    KUBECONFIG_CREDENTIAL_ID = 'kubeconfig-id'
    REGISTRY_CREDENTIAL_ID = 'Docker-credentials'
  }

  stages {
    // Stage to checkout code from SCM
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // Stage to build Docker image and push to registry
    stage('Build & Push Docker Image') {
      steps {
        script {
          // Get short git commit SHA for tagging
          def GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          // Compose full image tag
          def IMAGE_TAG = "${REGISTRY}/${IMAGE_NAME}:${GIT_SHA}"

          // Use Docker registry credentials to login, build, and push images
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

    // Stage to create TLS secret in Kubernetes
    // stage('Create TLS Secret') {
    //   steps {
    //     // Use credentials for TLS cert and key
    //     withCredentials([
    //       string(credentialsId: 'kube-worker-tls-crt', variable: 'TLS_CRT'),
    //       string(credentialsId: 'kube-worker-tls-key', variable: 'TLS_KEY')
    //     ]) {
    //       sh '''
    //         # Write TLS cert and key to files
    //         echo "$TLS_CRT" > tls.crt
    //         echo "$TLS_KEY" > tls.key

    //         # Create or update the TLS secret in Kubernetes namespace
    //         kubectl create secret tls tls-secret \
    //           --cert=tls.crt \
    //           --key=tls.key \
    //           -n $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

    //         # Clean up temporary files
    //         rm tls.crt tls.key
    //       '''
    //     }
    //   }
    // }

    stage('Create TLS Secret from Files') {
      environment {
        SECRET_NAME = 'tls-secret'
      }
      steps {
        // Load certificate and key as temporary files using Jenkins secret file credentials
        withCredentials([
          file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG'),
          file(credentialsId: 'kube-worker-ca', variable: 'TLS_CRT_FILE'),
          file(credentialsId: 'kube-worker-priv', variable: 'TLS_KEY_FILE')
        ]) {
          // sh '''
          //   echo "Using cert file: $TLS_CRT_FILE"
          //   echo "Using key file: $TLS_KEY_FILE"

          //   # Validate file content
          //   grep "BEGIN CERTIFICATE" "$TLS_CRT_FILE" || { echo "❌ Invalid TLS cert file format"; exit 1; }
          //   grep "BEGIN RSA PRIVATE KEY" "$TLS_KEY_FILE" || { echo "❌ Invalid TLS key file format"; exit 1; }

          //   # Create or update Kubernetes secret using the provided files
          //   // kubectl create secret tls $SECRET_NAME --cert=$TLS_CRT_FILE --key=$TLS_KEY_FILE -n $NAMESPACE --dry-run=client -o yaml
          //   echo "kubectl create secret tls \"$SECRET_NAME\" --cert=\"$TLS_CRT_FILE\" --key=\"$TLS_KEY_FILE\" -n \"$NAMESPACE\" --dry-run=client -o yaml"
          // '''
          sh '''
            echo "Debugging variable values:"
            echo "SECRET_NAME: $SECRET_NAME"
            echo "TLS_CRT_FILE: $TLS_CRT_FILE"
            echo "TLS_KEY_FILE: $TLS_KEY_FILE"
            echo "NAMESPACE: $NAMESPACE"

            echo "Expanded kubectl command:"
            echo "kubectl create secret tls \"$SECRET_NAME\" --cert=\"$TLS_CRT_FILE\" --key=\"$TLS_KEY_FILE\" -n \"$NAMESPACE\" --dry-run=client -o yaml"

            # Real command
            kubectl create secret tls "$SECRET_NAME" \
              --cert="$TLS_CRT_FILE" \
              --key="$TLS_KEY_FILE" \
              -n "$NAMESPACE" \
              --dry-run=client -o yaml | kubectl apply -f -
          '''
        }
      }
    }

    // Stage to install NGINX Ingress Controller if not present
    // stage('Install NGINX Ingress Controller') {
    //   steps {
    //     sh '''
    //       # Check if ingress-nginx namespace exists, create if not
    //       kubectl get namespace ingress-nginx || kubectl create namespace ingress-nginx

    //       # Deploy ingress-nginx controller manifest
    //       kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
    //     '''
    //   }
    // }
    stage('Install NGINX Ingress Controller') {
      steps {
        withCredentials([
          file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')
        ]) {
          sh '''
            echo "🔧 Installing NGINX Ingress Controller..."

            # Check cluster access
            kubectl cluster-info || { echo "❌ Unable to access cluster. Check kubeconfig."; exit 1; }

            # Check if the ingress-nginx namespace exists, create if not
            kubectl get namespace ingress-nginx || kubectl create namespace ingress-nginx

            # Apply the ingress-nginx controller manifest
            kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
          '''
        }
      }
    }

    // Stage to deploy the app to Kubernetes
    stage('Deploy to Kubernetes') {
      steps {
        script {
          def GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          def IMAGE_TAG = "${REGISTRY}/${IMAGE_NAME}:${GIT_SHA}"

          // Use kubeconfig credential file for kubectl commands
          withCredentials([file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
            sh """
              # Ensure namespace exists (create if needed)
              kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

              # Apply all k8s manifests from k8s directory
              kubectl apply -f k8s -n ${NAMESPACE}

              # Update deployment image to the new image tag
              kubectl set image deployment/${IMAGE_NAME} ${IMAGE_NAME}=${IMAGE_TAG} -n ${NAMESPACE}

              # Wait for rollout to complete
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
          // On failure, roll back to previous deployment image
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
    success {
      script {
        echo "Deployment successful!"
      }
    }
    always {
      script {
        echo "Cleaning up temporary files..."
        // Add any cleanup steps if necessary
      }
    }
  }

  // post {
  //   failure {
  //     script {
  //       withCredentials([file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
  //         def deploymentExists = sh(
  //           script: "kubectl -n ${NAMESPACE} get deployment ${IMAGE_NAME} --ignore-not-found",
  //           returnStatus: true
  //         ) == 0

  //         if (deploymentExists) {
  //           // Get container name dynamically
  //           def containerName = sh(
  //             script: "kubectl -n ${NAMESPACE} get deployment ${IMAGE_NAME} -o jsonpath='{.spec.template.spec.containers[0].name}'",
  //             returnStdout: true
  //           ).trim()

  //           def prevImage = sh(
  //             script: "kubectl -n ${NAMESPACE} get deployment ${IMAGE_NAME} -o jsonpath='{.spec.template.spec.containers[0].image}'",
  //             returnStdout: true
  //           ).trim()

  //           if (containerName && prevImage) {
  //             sh """
  //               kubectl -n ${NAMESPACE} set image deployment/${IMAGE_NAME} ${containerName}=${prevImage}
  //               kubectl -n ${NAMESPACE} rollout status deployment/${IMAGE_NAME}
  //             """
  //           } else {
  //             echo "Container name or previous image not found. Skipping rollback."
  //           }
  //         } else {
  //           echo "Deployment ${IMAGE_NAME} not found in namespace ${NAMESPACE}. Skipping rollback."
  //         }
  //       }
  //     }
  //   }
  //   success {
  //     script {
  //       echo "Deployment successful!"
  //     }
  //   }
  //   always {
  //     script {
  //       echo "Cleaning up temporary files..."
  //       // Add any cleanup steps if necessary
  //     }
  //   }
  // }
}
