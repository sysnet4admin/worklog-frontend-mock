pipeline {
    agent any

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
                    # ⚠️ frontend(Node/yarn)는 amd64 QEMU 에뮬레이션 빌드 시 Jenkins 에이전트(로컬 VM) 메모리
                    #    과부하로 빌드가 멈출 수 있다. VM 아키텍처에 맞춘 단일 플랫폼 사용을 권장한다.
                    #    (Apple Silicon=linux/arm64, Intel=linux/amd64) 멀티아치는 클라우드 러너(GitHub/GitLab)에서.
                    docker buildx build --platform linux/arm64 \\
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
