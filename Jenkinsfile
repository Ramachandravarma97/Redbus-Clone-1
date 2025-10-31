pipeline {
  agent any

  environment {
    REGISTRY      = "docker.io"
    DOCKERHUB_NS  = "ramachandravarma97"
    IMAGE_NAME    = "redbus-clone"
    IMAGE_TAG     = "${env.BUILD_NUMBER}"
    FULL_IMAGE    = "${REGISTRY}/${DOCKERHUB_NS}/${IMAGE_NAME}:${IMAGE_TAG}"
    AWS_REGION    = "ap-south-1"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -e
          docker build -t ${FULL_IMAGE} .
          docker image ls ${FULL_IMAGE}
        '''
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            set -e
            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin ${REGISTRY}
            docker push ${FULL_IMAGE}
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        // ✅ Use your AWS credential ID here (no need to add access/secret keys manually)
        withAWS(credentials: '2fd57bc0-0be1-44fc-acb6-9d7ccb278e6d', region: 'ap-south-1') {
          withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -e
              export KUBECONFIG="${KUBECONFIG_FILE}"
              export AWS_DEFAULT_REGION="${AWS_REGION}"

              # Validate AWS identity & cluster
              aws sts get-caller-identity
              kubectl cluster-info

              # Ensure namespace exists (safe if already present)
              kubectl apply -f k8s/namespace.yaml

              # Substitute image tag dynamically
              sed -e "s#ramachandravarma97/redbus-clone:\\${IMAGE_TAG}#${FULL_IMAGE}#g" \
                k8s/deployment.yaml > /tmp/deploy.yaml

              kubectl apply -f /tmp/deploy.yaml
              kubectl apply -f k8s/service.yaml

              kubectl -n redbus rollout status deploy/redbus-web --timeout=180s
              kubectl -n redbus get pods
              kubectl -n redbus get svc redbus-web
            '''
          }
        }
      }
    }
  }

  post {
    success { echo "✅ Deployment successful: ${FULL_IMAGE}" }
    failure { echo "❌ Deployment failed — check logs!" }
  }
}
