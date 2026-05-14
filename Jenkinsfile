pipeline {

    agent {
        label 'docker-agent'
    }

   stages {

        stage('Checkout Code') {

            steps {

                git branch:'main',url:'https://github.com/sharathg97/spring-petclinic.git'

            }
        }

        stage('Build Docker Image') {

            steps {

                sh """
                    docker build -t $IMAGE_NAME:$IMAGE_TAG .
                """

            }
        }

        stage('Tag Docker Image') {

            steps {

                sh """
                    docker tag $IMAGE_NAME:$IMAGE_TAG \
                    $JFROG_REGISTRY/$IMAGE_NAME:$IMAGE_TAG
                """

            }
        }

        stage('Login to JFrog') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'jfrog-creds',
                        usernameVariable: 'JFROG_USER',
                        passwordVariable: 'JFROG_PASS'
                    )
                ]) {

                    sh """
                        docker login $JFROG_REGISTRY \
                        -u $JFROG_USER \
                        -p $JFROG_PASS
                    """
                }
            }
        }

        stage('Push Image to JFrog') {

            steps {

                sh """
                    docker push \
                    $JFROG_REGISTRY/$IMAGE_NAME:$IMAGE_TAG
                """

            }
        }

        stage('Deploy to AKS') {

            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubernetes-aks',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh """
                        kubectl set image deployment/petclinic \
                        petclinic=$JFROG_REGISTRY/$IMAGE_NAME:$IMAGE_TAG \
                        -n $AKS_NAMESPACE
                    """

                }
            }
        }

        stage('Verify Deployment') {

            steps {

                sh """
                    kubectl rollout status deployment/petclinic
                """

            }
        }
    }

    post {

        success {

            echo 'Pipeline executed successfully.'

        }

        failure {

            echo 'Pipeline failed.'

        }

        always {

            sh 'docker system prune -f'

        }
    }
}
