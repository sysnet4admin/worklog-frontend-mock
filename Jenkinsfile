pipeline {
    agent any
    stages {
        stage('Say Hello') {
            steps {
                echo "Hello, Jenkins Pipeline!"
            }
        }
        stage('Show Workspace') {
            steps {
                sh 'ls -la'
            }
        }
    }
}
