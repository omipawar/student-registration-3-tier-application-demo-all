

### create .env file
````
VITE_API_URL = "http://a2f9ce90d8731479bb13ce1d48f20c94-203910496.ap-southeast-1.elb.amazonaws.com:8080/api" # replce backend-svc link
````
### edit .env file add backend svc link 
### build docker image
### push docker image
### edit manifest frontend.yaml -change image name
### apply service.yaml
### copy frontend link and see output
````
pipeline {
    agent any

    tools {
        nodejs 'node18'
        dockerTool 'docker'
    }

    environment {
        DOCKER_IMAGE = "abhipraydh96/b31-frontend"
        IMAGE_TAG    = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/abhipraydhoble/project-studentapp-three-tier-final.git'
            }
        }

        stage('Install & Build Frontend') {
            steps {
                dir('frontend') {
                    sh '''
                        npm install
                        npm run build
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir('frontend') {
                    sh """
                        docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }

       stage('Deploy to Kubernetes') {
    steps {
        withCredentials([
            file(credentialsId: 'config-file', variable: 'KUBECONFIG'),
            [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']
        ]) {
            sh """
                echo "Updating image in deployment.yaml"

                sed -i 's|image:.*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|' frontend/deployment.yaml

                kubectl apply -f frontend/deployment.yaml
                kubectl apply -f frontend/service.yaml
            """
         }
      }
   }

    }

}

````

````
pipeline {
    agent any

    tools {
        nodejs 'node18'
        dockerTool 'docker'
    }

    environment {
        DOCKER_IMAGE = "abhipraydh96/b31-frontend"
        IMAGE_TAG    = "${BUILD_NUMBER}"

        BACKEND_API_URL = "http://a2f9ce90d8731479bb13ce1d48f20c94-203910496.ap-southeast-1.elb.amazonaws.com:8080/api"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/abhipraydhoble/student-registration.git'
            }
        }

        stage('Create .env for Frontend') {
            steps {
                dir('frontend') {
                    sh """
                        echo "VITE_API_URL=${BACKEND_API_URL}" > .env
                        cat .env
                    """
                }
            }
        }

        stage('Install & Build Frontend') {
            steps {
                dir('frontend') {
                    sh '''
                        npm install
                        npm run build
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir('frontend') {
                    sh """
                        docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: 'config-file', variable: 'KUBECONFIG'),
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']
                ]) {
                    sh """
                        sed -i 's|image:.*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|' frontend/deployment.yaml

                        kubectl apply -f frontend/deployment.yaml
                        kubectl apply -f frontend/service.yaml

                        kubectl rollout status deployment/frontend
                    """
                }
            }
        }
    }
}

````

