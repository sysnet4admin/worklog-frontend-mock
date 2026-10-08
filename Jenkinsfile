pipeline {
    // 컨트롤러에는 docker가 없으므로 docker.sock을 가진 k8s 에이전트(JCasC kubernetes cloud)에서 실행.
    // agent any면 컨트롤러 실행기(numExecutors: 2)로 갈 수 있고 그때 'docker: not found'로 실패한다.
    agent { label 'jenkins-jenkins-agent' }

    environment {
        DOCKER_REPOSITORY = 'worklog-frontend-mock'
        // 클러스터 노드(cp-k8s, w1~w3)가 모두 arm64(Apple Silicon 호스트)라 단일 플랫폼으로 빌드한다.
        // Intel 호스트라면 linux/amd64로 바꾼다. 멀티아치는 QEMU 에뮬레이션으로 yarn build가 돌아
        // 로컬 에이전트 메모리가 부족해지므로 쓰지 않는다.
        BUILD_PLATFORM = 'linux/arm64'
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        GITHUB_CREDENTIALS = credentials('github-token')
    }

    stages {
        stage('Init') {
            steps {
                script {
                    env.SHORT_SHA = sh(script: 'git rev-parse --short=8 HEAD', returnStdout: true).trim()
                }
                echo "Image tag: ${env.SHORT_SHA}"
            }
        }

        stage('Build Image') {
            steps {
                sh """
                    docker buildx rm frontend-builder 2>/dev/null || true
                    docker buildx create --name frontend-builder --driver docker-container --use
                    echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login --username ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                    docker buildx build --platform ${BUILD_PLATFORM} \\
                        -t ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_REPOSITORY}:${env.SHORT_SHA} \\
                        --push .
                """
                echo "Built: ${DOCKERHUB_CREDENTIALS_USR}/${DOCKER_REPOSITORY}:${env.SHORT_SHA}"
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
        always { sh 'docker buildx rm frontend-builder 2>/dev/null || true' }
    }
}
