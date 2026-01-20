**Jenkins**
````
sudo apt update
sudo apt install fontconfig openjdk-21-jre  -y
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
````
**Docker**
````

sudo apt-get update
sudo apt-get install docker.io -y
sudo systemctl start docker
sudo usermod -aG docker ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock
````
**SonarQube**
````
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
````
##  Connect to Jenkins 

## Install Required Plugins:
   **Install below plugins**

````
Eclipse Temurin Installer 
````
````
SonarQube Scanner
````
````
NodeJs Plugin
````
````
docker
````
````
stage view
````
## Install  Tools: Manage Jenkins->Tools
   - add jdk: "jdk17" ->install from adoptium.net->version- 17
   - add SonarQube Scanner: "sonar-scanner"
   - add NodeJs: "node18" -> version 18.15.1
   - docker: "docker"

### **Configure Java and Nodejs in Global Tool Configuration**
Goto Manage Jenkins → Tools → Install JDK(17) and NodeJs(16)→ Click on Apply and Save
#### Jdk
![image](https://github.com/user-attachments/assets/fe876745-d024-403c-806b-4a7d8c1dba11)
#### SonarQube-Scanner
![image](https://github.com/user-attachments/assets/24589963-9a7e-4d6a-9598-66580c195e30)

#### Node-js
![image](https://github.com/user-attachments/assets/51617874-be4d-438c-a93e-5a5d9e5781fa)
#### Docker
![image](https://github.com/user-attachments/assets/289c2e2a-df33-476b-a195-d584db3ef03e)

##  Log in to Sonarqube and generate token
 - username: admin
 - password: admin
<img width="1902" height="957" alt="image" src="https://github.com/user-attachments/assets/36620768-5f81-440c-b31b-ecf29c609f64" />   

##  Add DockerHub & Sonarqube Credentials:
   **Docker**
  - Go to  "Manage Jenkins" → Credentials."
  - Click on "Global."
  - Click on "Add Credentials" 
  - Choose "username with password" as the kind of credentials.
  - Enter your DockerHub credentials (Username and Password) and give the credentials an ID (e.g., "docker-cred").
  - Click "OK" to save your DockerHub credentials.
    
     **SonarQube**
  - Go to  "Manage Jenkins" → Credentials."
  - Click on "Global."
  - Click on "Add Credentials" 
  - Choose "secret text" as the kind of credentials.
  - Enter your sonarqube token and give the credentials an ID (e.g., "sonar-token").
  - Click "create" to save yourcredentials

<img width="1907" height="846" alt="image" src="https://github.com/user-attachments/assets/bcd447f5-4a49-478d-99d6-1379202f4334" />

##  Configure Sonar Server: Manage Jenkins->System
   - name: "sonar-server"
   - url:
   - token:
![image](https://github.com/user-attachments/assets/c5d05628-1502-4a92-b722-7ad3eed5d587)

## Configure sonarqube webhook
  - go to sonarqube-->administration--->configuration-->webhook

##  Create Pipeline

```jenkinsfile
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven'
        dockerTool 'docker'
    }

    environment {
        DOCKER_IMAGE = "abhipraydh96/std-test-be"
        IMAGE_TAG    = "${BUILD_NUMBER}"
    }

    stages {

        stage('Code-Pull') {
            steps {
                git branch: 'main', url: 'https://github.com/abhipraydhoble/project-studentapp-three-tier-final.git'
            }
        }

        stage('Build') {
            steps {
                dir('backend') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        

        stage('Docker-Build') {
            steps {
                dir('backend') {
                    sh """
                      docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Docker-Push') {
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

       stage('Deploy') {
    steps {
        withCredentials([
            file(credentialsId: 'config-file', variable: 'KUBECONFIG'),
            [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']
        ]) {
            sh """
                sed -i 's|image:.*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|' backend/deployment.yaml

                kubectl apply -f backend/configmap.yaml
                kubectl apply -f backend/secret.yaml
                kubectl apply -f backend/deployment.yaml
                kubectl apply -f backend/service.yaml
            """
        }
    }
}

    }
}
```

````
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven'
    }

    environment {
        DOCKER_IMAGE = "abhipraydh96/std-test-be"
        IMAGE_TAG    = "${BUILD_NUMBER}"
        KUBE_NAMESPACE = "default"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/abhipraydhoble/project-studentapp-three-tier-final.git'
            }
        }

        stage('Build Backend') {
            steps {
                dir('backend') {
                    sh 'mvn clean verify -DskipTests'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('backend') {
                    withSonarQubeEnv('sonarqube') {
                        sh """
                            mvn sonar:sonar \
                              -Dsonar.projectKey=student-backend
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('backend') {
                    sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-cred',
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

        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    file(credentialsId: 'config-file', variable: 'KUBECONFIG'),
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']
                ]) {
                    sh """
                        sed -i 's|image: .*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|' backend/deployment.yaml

                     
                        kubectl apply -n ${KUBE_NAMESPACE} -f backend/secret.yaml
                        kubectl apply -n ${KUBE_NAMESPACE} -f backend/deployment.yaml
                        kubectl apply -n ${KUBE_NAMESPACE} -f backend/service.yaml

                     
                    """
                }
            }
        }
    }

}
````




