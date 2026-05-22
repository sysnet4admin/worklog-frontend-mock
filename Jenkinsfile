pipeline {
    agent any
    environment {
        DOCKER_REPOSITORY = 'sysnet4admin/worklog-frontend-mock'
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
                    docker buildx rm mybuilder 2>/dev/null || true
                    docker buildx create --name mybuilder --driver docker-container --use
                    docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
                    docker buildx build \\
                        --platform linux/amd64,linux/arm64 \\
                        -t ${DOCKER_REPOSITORY}:${SHORT_SHA} \\
                        --push .
                """
            }
        }
        stage('Update Manifest') {
            steps {
                sh """
                    sed -i 's|image: .*worklog-frontend-mock:.*|image: ${DOCKER_REPOSITORY}:${SHORT_SHA}|' deploy_manifest/worklog-frontend.yaml
                    git config user.name "jenkins"
                    git config user.email "jenkins@myk8s.local"
                    git remote set-url origin https://${GITHUB_CREDENTIALS_USR}:${GITHUB_CREDENTIALS_PSW}@github.com/sysnet4admin/worklog-frontend-mock.git
                    git add deploy_manifest/
                    git commit -m "deploy: update frontend image tag to ${SHORT_SHA}"
                    git push origin HEAD:main
                """
            }
        }
    }
    post {
        success { echo 'Frontend pipeline succeeded!' }
        failure { echo 'Frontend pipeline failed!' }
    }
}
