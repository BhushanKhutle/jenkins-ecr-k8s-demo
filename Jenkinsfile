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
    APP_NAME = "demo-nginx"
    NAMESPACE = "demo"
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
            docker build -t $APP_NAME:$BUILD_NUMBER .
          '''
        }
      }
    }

    stage("Login ECR + Push") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          container('docker') {
            sh '''
              apk add --no-cache aws-cli
              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

              docker tag $APP_NAME:$BUILD_NUMBER $IMAGE:$BUILD_NUMBER
              docker push $IMAGE:$BUILD_NUMBER
            '''
          }
        }
      }
    }

    stage("Deploy to Kubernetes") {
      steps {
        container('kubectl') {
          sh '''
            kubectl create ns $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

            kubectl -n $NAMESPACE set image deploy/$APP_NAME $APP_NAME=$IMAGE:$BUILD_NUMBER --record || \
            kubectl -n $NAMESPACE create deploy $APP_NAME --image=$IMAGE:$BUILD_NUMBER

            kubectl -n $NAMESPACE rollout status deploy/$APP_NAME
          '''
        }
      }
    }
  }
}
