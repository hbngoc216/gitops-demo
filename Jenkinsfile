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

    triggers {
        githubPush()
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

        stage('Check Commit') {
            steps {
                script {
                    def commitMessage = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()

                    echo "Commit message: ${commitMessage}"

                    if (commitMessage.contains('[skip ci]')) {
                        currentBuild.result = 'NOT_BUILT'
                        error('Skipping Jenkins-generated GitOps commit')
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                container('docker') {
                    retry(3) {
                        sh '''
                            set -e

                            echo "Building image: $IMAGE:$TAG"

                            docker version
                            docker pull nginx:alpine
                            docker build -t "$IMAGE:$TAG" .
                        '''
                    }
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
                            set -e

                            echo "$DOCKER_PASS" |
                              docker login \
                                -u "$DOCKER_USER" \
                                --password-stdin

                            docker push "$IMAGE:$TAG"

                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Update Helm') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e

                        echo "Updating Helm image tag to $TAG"

                        sed -i -E \
                          's/^([[:space:]]*tag:).*/\\1 "'"${TAG}"'"/' \
                          chart/values.yaml

                        echo "Updated chart/values.yaml:"
                        grep -A4 '^image:' chart/values.yaml

                        git config user.email "jenkins@lab.local"
                        git config user.name "jenkins"

                        git add chart/values.yaml

                        git commit \
                          -m "Update image ${TAG} [skip ci]" \
                          || echo "No changes to commit"

                        git remote set-url origin \
                          "https://${GIT_USER}:${GIT_TOKEN}@github.com/hbngoc216/gitops-demo.git"

                        git push origin HEAD:main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Image pushed: ${IMAGE}:${TAG}"
        }

        failure {
            echo "Pipeline failed. Check the failed stage in Console Output."
        }

        always {
            echo "Build number: ${BUILD_NUMBER}"
        }
    }
}
