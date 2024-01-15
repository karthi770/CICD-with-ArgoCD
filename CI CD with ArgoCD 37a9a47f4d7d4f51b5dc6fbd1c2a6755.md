# CI/CD with ArgoCD

## EC2 instance creation:

Create an EC2 instance with t2.large and setup the security groups.

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled.png)

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%201.png)

Create an IAM role with Admin access and attach it to the EC2 instance.

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%202.png)

Install docker in the EC2 instance

```bash
sudo apt install [docker.io](http://docker.io/)
```

```bash
usermod -aG docker jenkins
usermod -aG docker ubuntu
systemctl daemon-reload
systemctl restart docker
```

Install Sonarqube in EC2 instance:

```bash
apt install unzip
adduser sonarqube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
unzip *
chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```

## Local Installations: Minikube and Argo CD:

Minikube installation

```json
curl -LO [https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64](https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64)
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

```bash
sudo snap install kubectl --classic
```

```bash
minikube start
```

Argo CD operators:

Install Operator Lifecycle Manager (OLM), a tool to help manage the Operators running on your cluster.

```bash
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.25.0/install.sh | bash -s v0.25.0
```

Install the operator by running the following command:

```bash
kubectl create -f https://operatorhub.io/install/argocd-operator.yaml
```

After install, watch your operator come up using next command.

```bash
kubectl create -f https://operatorhub.io/install/argocd-operator.yaml
```

## Configure Credentials and Git repo:

Configure jenkins to the Git repo

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%203.png)

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%204.png)

Inside Sonarqube generate a token which can be used inside jenkins in order to authenticate.

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%205.png)

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%206.png)

Setup docker credentials

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%207.png)

Generate token for Github access

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%208.png)

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%209.png)

## Jenkins Pipeline:

Stage 1: Git checkout

In the first stage the maven has been installed. A docker image has been used to install maven. Also the git checkout is carried out in this stage.

```bash
pipeline {
  agent {
    docker {
      image 'abhishekf5/maven-abhishek-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock' // mount Docker socket to access the host's Docker daemon
    }
  }
```

```bash
stages {
    stage('Checkout') {
      steps {
        sh 'echo passed'
        //git branch: 'main', url: 'https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero.git'
      }
    }
```

Stage 2: Build and Test

```bash
stage('Build and Test') {
      steps {
        sh 'ls -ltr'
        // build the project and create a JAR file
        sh 'cd spring-boot-app && mvn clean package'
      }
    }
```

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2010.png)

Stage 3: Static Code Analysis

```bash
stage('Static Code Analysis') {
      environment {
        SONAR_URL = "http://184.73.122.112:9000"
      }
      steps {
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
          sh 'cd spring-boot-app && mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
        }
      }
    }
```

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2011.png)

Stage 4: Docker Build and Image push:

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2012.png)

```bash
stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "karthi770/spring-boot-app:${BUILD_NUMBER}"
        // DOCKERFILE_LOCATION = "java-maven-sonar-argocd-helm-k8s/spring-boot-app/Dockerfile"
        REGISTRY_CREDENTIALS = credentials('docker-cred')
      }
      steps {
        script {
            sh 'cd spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
            def dockerImage = docker.image("${DOCKER_IMAGE}")
            docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                dockerImage.push()
            }
        }
      }
    }
```

Following code is used in the Docker File:

```bash
FROM adoptopenjdk/openjdk11:alpine-jre

ARG artifact=target/spring-boot-web.jar

WORKDIR /opt/app

COPY ${artifact} app.jar

ENTRYPOINT ["java","-jar","app.jar"]
```

![Image has been uploaded in the dockerhub.](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2013.png)

Image has been uploaded in the dockerhub.

Stage 5: Deployment

```bash
stage('Update Deployment File') {
        environment {
            GIT_REPO_NAME = "cicd_ArgoCD"
            GIT_USER_NAME = "karthi770"
        }
        steps {
            withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                sh '''
                    git config user.email "karthikeyanselvaraj37@gmail.com"
                    git config user.name "karthi770"
                    BUILD_NUMBER=${BUILD_NUMBER}
                    sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" spring-boot-app-manifests/deployment.yml
                    git add spring-boot-app-manifests/deployment.yml
                    git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                    git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                '''
            }
        }
    }
```

Update the deployment.yml file:

```bash
spec:
      containers:
      - name: spring-boot-app
        image: karthi770/spring-boot-app:replaceImageTag
        ports:
        - containerPort: 8080
```

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2014.png)

## CD process with ArgoCD:

Start with checking the minikube status

```bash
minikube status
```

create a .yml file named argocd-basics

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
  labels:
    example: basic
spec: {}
```

Custom Resource Definition (CRD) for the kind "ArgoCD" in the specified version "[argoproj.io/v1alpha1](http://argoproj.io/v1alpha1)". This typically happens when the necessary CRDs for ArgoCD are not installed in your cluster before applying the resource manifest.

```yaml
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/application-crd.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/application-crd.yaml)
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/appproject-crd.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/appproject-crd.yaml)
```

```yaml
kubectl get crd -n argocd
```

```yaml
kubectl apply -f argocd-basics.yml
```

```yaml
kubectl get svc
```

You will get output as : example-argocd-server

```yaml
kubectl edit svc example-argocd-server
```

```yaml
minikube service example-argocd-server
```

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2015.png)

```yaml
kubectl get secret
```

```yaml
kubectl edit secret example-argocd-cluster
```

```yaml
echo -n NVdSWFF2SjdQdHIyZXhIRndWbVNraE5mOGkxYkRneUw= | base64 -d 
```

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2016.png)

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2017.png)

![Put namespace as default.](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2018.png)

Put namespace as default.

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2019.png)

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2020.png)

![Untitled](CI%20CD%20with%20ArgoCD%2037a9a47f4d7d4f51b5dc6fbd1c2a6755/Untitled%2021.png)