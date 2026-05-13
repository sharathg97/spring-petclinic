pipeline {

    agent {
        label 'docker-agent'
    }

    stages {

        stage('Check Docker') {

            steps {
                sh 'docker --version'
            }
        }
    }
}
