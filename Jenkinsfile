pipeline {
    agent {
        label 'docker-agent'
    }

    environment {
        APP_NAME = "spring-petclinic"

        //noinspection HttpUrlsUsage
        JFROG_URL = "http://52.229.154.181:8082/artifactory"
        JFROG_REPO = "jfrog-springpetclinic"

        // Jar details
        JAR_FILE = "target/spring-petclinic-4.0.0-SNAPSHOT.jar"
        ARTIFACT_PATH = "com/petclinic/${BUILD_NUMBER}/spring-petclinic.jar"

        // AKS
        AKS_NAMESPACE = "petclinic"

        // Docker image
        IMAGE_NAME = "petclinic-app"
        IMAGE_TAG = "${BUILD_NUMBER}"

        // Azure Credentials
        AZURE_CLIENT_ID     = credentials('azure-client-id')
        AZURE_CLIENT_SECRET = credentials('azure-client-secret')
        AZURE_TENANT_ID     = credentials('azure-tenant-id')
        AZURE_SUBSCRIPTION  = credentials('azure-subscription-id')

        // Azure Container Registry
        ACR_NAME = "sharacrcontreg"
    }

    stages {

        // ---------------- CHECKOUT ----------------
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/sharathg97/spring-petclinic.git'
            }
        }

        // ---------------- MAVEN BUILD ----------------
        stage('Maven Build') {
            steps {
                sh '''
                   mvn clean package -DskipTests -Dcheckstyle.skip=true
                '''
            }
        }

        // ---------------- VERIFY ARTIFACT ----------------
        stage('Verify Artifact') {
            steps {
                sh '''
                    echo "Checking generated artifact..."
                    ls -lh target/
                '''
            }
        }

        // ---------------- UPLOAD JAR TO JFROG ----------------
        stage('Upload JAR to JFrog Artifactory') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'jfrog-creds',
                        usernameVariable: 'JFROG_USER',
                        passwordVariable: 'JFROG_PASS'
                    )
                ]) {

                    sh '''
                        echo "Uploading JAR to JFrog..."

                        curl -u $JFROG_USER:$JFROG_PASS \
                        -T $JAR_FILE \
                        "$JFROG_URL/$JFROG_REPO/$ARTIFACT_PATH"
                    '''
                }
            }
        }

        // ---------------- AZURE LOGIN ----------------
        stage('ACR Login') {
            steps {

                sh '''
                    echo "Logging into Azure..."

                    az login --service-principal \
                      -u $AZURE_CLIENT_ID \
                      -p $AZURE_CLIENT_SECRET \
                      --tenant $AZURE_TENANT_ID

                    az account set --subscription $AZURE_SUBSCRIPTION

                    echo "Logging into Azure Container Registry..."

                    az acr login --name $ACR_NAME
                '''
            }
        }

        // ---------------- BUILD DOCKER IMAGE ----------------
        stage('Docker Build') {
            steps {

                sh '''
                    echo "Building Docker image..."

                    docker build --no-cache \
                      -t $ACR_NAME.azurecr.io/$IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        // ---------------- PUSH IMAGE ----------------
        stage('Push Image') {
            steps {

                sh '''
                    echo "Pushing Docker image..."

                    docker push $ACR_NAME.azurecr.io/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        // ---------------- DEPLOY TO AKS ----------------
        stage('Deploy to AKS') {
            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubernetes-aks',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        echo "Injecting image tag into deployment file..."

                        sed "s|IMAGE_TAG|$IMAGE_TAG|g" \
                        k8s/dev/deployment.yaml > k8s/dev/deployment-temp.yaml

                        echo "Final Deployment Manifest:"
                        cat k8s/dev/deployment-temp.yaml

                        echo "Deploying application to AKS..."

                        kubectl apply -f k8s/dev/deployment-temp.yaml

                        kubectl apply -f k8s/dev/service.yaml

                        echo "Waiting for deployment rollout..."

                        kubectl rollout status \
                        deployment/spring-petclinic-app \
                        --timeout=5m || true

                        echo "Pods Status:"
                        kubectl get pods -n $AKS_NAMESPACE -o wide

                        echo "Services Status:"
                        kubectl get svc -n $AKS_NAMESPACE
                    '''
                }
            }
        }
    }

    post {

    success {
        echo 'Pipeline executed successfully.'

        mail to: 'sharathgwork@gmail.com',
             subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
             body: """
Hello Team,

The Jenkins pipeline completed successfully.

Job Name   : ${env.JOB_NAME}
Build Number : ${env.BUILD_NUMBER}
Build URL  : ${env.BUILD_URL}

Status     : SUCCESS

Regards,
Jenkins
"""
    }

    failure {
        echo 'Pipeline execution failed.'

        mail to: 'sharathgwork@gmail.com',
             subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
             body: """
Hello Team,

The Jenkins pipeline execution failed.

Job Name   : ${env.JOB_NAME}
Build Number : ${env.BUILD_NUMBER}
Build URL  : ${env.BUILD_URL}

Status     : FAILED

Please check the console logs.

Regards,
Jenkins
"""
    }

    always {

        sh '''
            echo "Cleaning workspace..."

            rm -rf target/*.jar || true
        '''
    }
}
}
