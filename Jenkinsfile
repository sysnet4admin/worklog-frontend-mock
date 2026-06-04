pipeline {
    agent {
        kubernetes {
            yaml '
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: sungminl/inbound-agent-docker:3248.v65ecb_254c298-3-jdk17
    securityContext:
      runAsUser: 0
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
'
        }
    }

    environment {
        DOCKER_REPOSITORY = 'worklog-frontend-mock'
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        GITHUB_CREDENTIALS = credentials('github-token')
    }

    stages {
        stage('Init') {
            steps {
                script {
                    env.SHORT_SHA = sh(script: 'git rev-parse --short=8 HEAD', returnStdout: true).trim()
                }
            }
        }

        stage('Build Image') {
            steps {
                sh """
                    docker run --privileged --rm tonistiigi/binfmt --install all 2>/dev/null || true
                    docker buildx rm frontend-builder 2>/dev/null || true
                    docker buildx create --name frontend-builder --driver docker-container --use
                    echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login --username ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                    docker buildx build --platform linux/amd64,linux/arm64 \\
                        -t ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_REPOSITORY}:${env.SHORT_SHA} \\
                        --push .
                """
                echo "Built: ${env.SHORT_SHA}"
            }
        }

        stage('Update Manifest') {
            steps {
                sh """
                    sed -i "s|image: .*/worklog-frontend-mock:.*|image: ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_REPOSITORY}:${env.SHORT_SHA}|" deploy_manifest/worklog-frontend.yaml
                    git config user.name "jenkins"
                    git config user.email "jenkins@myk8s.local"
                    git remote set-url origin https://${GITHUB_CREDENTIALS_USR}:${GITHUB_CREDENTIALS_PSW}@github.com/${GITHUB_CREDENTIALS_USR}/worklog-frontend-mock.git
                    git add deploy_manifest/
                    git diff --staged --quiet || git commit -m "deploy: update frontend image to ${env.SHORT_SHA}"
                    git pull --rebase origin main || true
                    git push origin HEAD:main
                """
                echo "Deployed: ${env.SHORT_SHA}"
            }
        }
    }

    post {
        success { echo "Frontend pipeline succeeded: ${env.SHORT_SHA}" }
        failure { echo "Frontend pipeline failed" }
    }
}
