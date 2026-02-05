pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: kubectl
    image: dtzar/helm-kubectl:3.15.4
    command: ["cat"]
    tty: true

  - name: docker
    image: docker:24.0.7
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock

  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
"""
    }
  }

  environment {
    AWS_REGION = "ap-south-1"
    ECR_REGISTRY = "231907690017.dkr.ecr.ap-south-1.amazonaws.com"
    ECR_REPO = "demo-nginx"
    IMAGE = "${ECR_REGISTRY}/${ECR_REPO}"
    NAMESPACE = "demo"
    RELEASE_NAME = "demo-nginx"
  }

  stages {

    stage("Checkout") {
      steps {
        checkout scm
      }
    }

    stage("Build Docker Image") {
      steps {
        container('docker') {
          sh '''
            docker build -t demo-nginx:$BUILD_NUMBER .
          '''
        }
      }
    }

    stage("Login ECR & Push Image") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds',
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {

          container('docker') {
            sh '''
              apk add --no-cache aws-cli

              aws ecr get-login-password --region $AWS_REGION \
              | docker login --username AWS --password-stdin $ECR_REGISTRY

              docker tag demo-nginx:$BUILD_NUMBER $IMAGE:$BUILD_NUMBER
              docker push $IMAGE:$BUILD_NUMBER
            '''
          }
        }
      }
    }

    stage("Deploy using Helm") {
      steps {
        container('kubectl') {
          sh '''
            kubectl create ns $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

            helm upgrade --install $RELEASE_NAME ./helm/demo-nginx \
              -n $NAMESPACE \
              --set image.tag=$BUILD_NUMBER
          '''
        }
      }
    }

  }
}

