pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: docker
      image: docker:27-cli
      command:
        - cat
      tty: true
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock

  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
        type: Socket
'''
        }
    }

    environment {
        IMAGE = "docker.io/hbnu/gitops-demo"
        TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                container('docker') {
                    sh '''
                        docker version
                        docker build -t $IMAGE:$TAG .
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                container('docker') {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh '''
                            echo "$DOCKER_PASS" \
                              | docker login -u "$DOCKER_USER" --password-stdin

                            docker push $IMAGE:$TAG
                        '''
                    }
                }
            }
        }

        stage('Update Helm') {
            steps {
                sh '''
                    sed -i -E 's/^([[:space:]]*tag:).*/\\1 "'"$TAG"'"/' chart/values.yaml

                    git config user.email "jenkins@lab.local"
                    git config user.name "jenkins"

                    git add chart/values.yaml
                    git commit -m "Update image ${TAG} [skip ci]" || true
                '''
            }
        }
    }
}
