pipeline {

    agent {
        label 'docker-agent'
    }
environment {

        IMAGE_NAME = "petclinic-app"
        IMAGE_TAG = "${BUILD_NUMBER}"

        JFROG_REGISTRY = "52.229.154.181:8082/artifactory/docker-local"

        AKS_NAMESPACE = "petclinic"

    }
   stages {

        stage('Checkout Code') {

            steps {

                git branch:'main',url:'https://github.com/sharathg97/spring-petclinic.git'

            }
        }

       stage('Maven Build ') {

            steps {

                sh """
                    mvn clean install -DskipTests
                """

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
                        docker login 52.229.154.181:8082 \
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

        
    }
}
