# In this project, we will Deploy a springboot Java Application into an EKS (Elastic Kubernetes Service) cluster using Helm and a Jenkins Pipeline 

![diagram](./images/awsoverview.jpg)

![](https://i.imgur.com/waxVImv.png)
### [View all Roadmaps](https://github.com/nholuongut/all-roadmaps) &nbsp;&middot;&nbsp; [Best Practices](https://github.com/nholuongut/all-roadmaps/blob/main/public/best-practices/) &nbsp;&middot;&nbsp; [Questions](https://www.linkedin.com/in/nholuong/)
<br/>

Our pipeline workflow will be as follows:

1. Push code into Version Control System (Github) - Application code, Jenkinsfile,Dockerfile and Helm Chart
2. Pushing to Github triggers Jenkins pipeline
3. Jenkins checks out the code from Github
4. Maven builds JAR file
5. Unit tests are performed
6. Build Docker Image with dockerfile
7. Tag and push Docker Image to ECR
8. Deploy to EKS using Helm



Pre-requisites

1. EKS Cluster - For simplicity, we shall use eksctl to create cluster with 2 worker nodes

2. ECR Repository - Create one on AWS. This will host our Docker image.

3. Launch EC2 to act as our Jenkins server
- Install the following on our Jenkins server
   a. AWSCLI
   b. Helm
   c. Kubectl
   d. Docker
   e. eksctl
   f. Jenkins
   g. Maven


IMPLEMENTATION STEPS ARE AS FOLLOWS
- Create Github repo and constantly push our code from VSCode to Github repo (Application code, Jenkinsfile,Dockerfile and Helm Chart)
- Create Jenkins Server (EC2) and install all the necessary tools and applications as above
- Create an EKS cluster with 2 worker nodes in AWS using eksctl
- Create an ECR repository to host our docker image
- Create a Helm chart for our springboot application
- Update Helm chart to pull docker image in ECR
- Create kubeconfig credentials in Jenkins to interact with EKS cluster 
- Create Jenkins pipeline script for jenkinsfile
- Configure Jenkins to use Jenkinsfile from SCM
- Create Namespace for Helm deployment
- Deploy application into namespace in EKS
- Access application in browser using LoadBalancer url




CREATE JENKINS SERVER

Launch EC2

Name: Jenkins Server

AMI: Ubuntu 22.04 LTS

INSTANCE TYPE: t2.Large  (This isn't on the free tier and will cost you. We need enough RAM and CPU to install the pre-requisites) 

![Launch EC2](./images/launch-ec2.png)


INSTALLATIONS ON EC2

INSTALL AWSCLI

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" 

sudo apt install unzip

sudo unzip awscliv2.zip  

sudo ./aws/install

aws --version
```
![aws cli](./images/aws-cli.png)


INSTALL HELM
```
sudo apt update

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

chmod +x get_helm.sh

sudo ./get_helm.sh
```
Check helm version

```
helm version
```

![Helm](./images/helm.png)

INSTALL KUBECTL

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo touch /etc/apt/sources.list.d/kubernetes.list

echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

sudo apt-get install -y kubectl
```

VERIFY INSTALLATION

```
kubectl version --short --client
```

![kubectl](./images/kubectl.png)

INSTALL EKSCTL

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
Move the extracted binary to /usr/local/bin. 
```
sudo mv /tmp/eksctl /usr/local/bin
```
VERIFY INSTALLATION
```
eksctl version
```
![eksctl](./images/eksctl.png)


INSTALL DOCKER

Update local packages 
```
sudo apt update
```
```
Install the below packages```
sudo apt install gnupg2 pass -y
```
 
Install docker
```
sudo apt install docker.io -y
```

Add Ubuntu user to Docker group
```
sudo usermod -aG docker $USER
```
We need to reload shell in order to have new group settings applied. 
```
newgrp docker
```
The Docker service needs to be setup to run at startup. 
```
sudo systemctl start docker
sudo systemctl enable docker
```

Check the installation
```
sudo systemctl status docker
```
![docker](./images/docker.png)

INSTALL JENKINS
```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key |sudo gpg --dearmor -o /usr/share/keyrings/jenkins.gpg

sudo sh -c 'echo deb [signed-by=/usr/share/keyrings/jenkins.gpg] http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

sudo apt update

sudo apt install jenkins

sudo systemctl start jenkins.service
```

Check if Jenkins in running
```
sudo systemctl status jenkins
```
![jenkins](./images/jenkins.png)


Access your jenkins istallation in your web browser using your instance public IP address on port 8080
Make sure that port 8080 is open in your inbound rules for your Jenkins server  EC2 Instance

Go to http://yourinstancepublicip:8080

Open Port 8080 from everywhere on your instance if you want to expose it to everyone during testing
![jankins](./images/jenkins4.png)

Run cat /var/lib/jenkins/secrets/initialAdminPassword to get the initial login password

![jankins](./images/jenkins5.png)
![jankins](./images/jenkins6.png)
![jankins](./images/jenkins7.png)
![jankins](./images/jenkins8.png)
![jankins](./images/jenkins9.png)
![jankins](./images/jenkins10.png)




INSTALL DOCKER AND DOCKER PIPELINE PLUGIN AND ADD JENKINS USER TO DOCKER GROUP

```

Now Login to Jenkins > manage Jenkins > Available plugins and install
- Docker plugin
- Docker Pipeline plugin
```

![jankins](./images/jenkins11.png)

![docker plugins](./images/docker-install.png)


Add jenkins user to Docker group
```
sudo usermod -a -G docker jenkins
```
Restart Jenkins service
```
sudo service jenkins restart
```
Reload system daemon files
```
sudo systemctl daemon-reload
```

Restart Docker service 
```
sudo service docker stop
sudo service docker start
```
Check the Docker status
```
sudo systemctl status docker
```
![docker](./images/docker.png)



LETS CONFIGURE MAVEN PLUGIN THAT WILL BE USED IN OUR JENKINS PIPELINE

- We shall be using Maven for our build

Manage Jenkins > Tools > Maven > Add maven > Save

Go back into Manage Jenkis > Tools > Maven Installations
Set Maven name as Maven 3 and MAVEN_HOME as /usr/share/maven

![maven](./images/maven.png)



CREATE HELM CHART AND MODIFY VALUES.YAML AND DEPLOYMENT.YAML

Let's go to the root of our springboot application omn our local machine. 

Create helm chart by executing the commands below
```
helm create mychart
```
```
tree mychart
```

Add Docker image details to download from ECR before deploying to EKS cluster
open mychart/values.yaml.



Add Docker image details to download image from ECR before deploying to EKS cluster
open mychart/values.yaml.

Edit repository : ImageName

![values1](./images/values1.png)

Enter the service type as LoadBalancer

![values2](./images/values2.png)

Next, open templates/deployment.yaml
and update containerPort to 8080 as springboot application



![port](./images/port.png)

#### Save the files and push to our Github repo

![git](./images/git.png)


![github](./images/github.png)





CREATE EKS CLUSTER USING EKSCTL
#### We need to create Create IAM Role with Administrator Access

- Create an IAM role with AdministratorAccess policy.
- Go to AWS console > IAM > Roles. create a role
- Select AWS services, Click EC2, Click on Next permissions.
AdministratorAccess policy and select
- Name your role and create it

#### Next, we need to assign the created role to our Jenkins instance

 - Go to AWS console > EC2 > EC2 instance > Security

 - Click on Modify IAM Role

 - Choose the role we created from the dropdown and click on Apply.

Our Instance can now request services on our behalf

CREATE EKS CLUSTER WITH eksctl

Switch to Jenkins user
```
sudo su - jenkins
```
Run command to create EKS cluster with 2 worker nodes
```
eksctl create cluster --name demo-eks --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --managed --nodes 2 
```
Verify the cluster has been created by viewing the 2 worker nodes. We have already installed Kubectl on the Jenkins instance.

```
kubectl get nodes
```
![get nodes](./images/get-nodes.png)

Check the EC2 console. We now have our Jenkins server and the 2 worker nodes running.
![get nodes](./images/get-nodes2.png)

Before we deploy our application using helm, we want to create a namespace and deploy our application into that namespace.

```
create namespace helm-deployment
```


We shall the deploy into the namespace using the hel upgrade install command in our pipelines deploy stage. 

CREATE JENKINS PIPELINE

Dashboard > New Item > Enter Item name > Select Pipeline and click ok.

![new item](./images/jenkins-new-item.png)

Let's configure the pipeline to use the Jenkinsfile from our github repository

![git](./images/pipeline-configure.png)


#### Jenkinsfile

Here are our Jenkins pipeline stages

We have created an environment variable for our ECR repository url
```
    environment {
        registry = "185439933271.dkr.ecr.us-east-1.amazonaws.com/my-ecr-repo"
    }
```

1. Checkout code from Github

```
        stage('Git Checkout') {
            steps {
               checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/deleonab/cicd-jenkins-docker-helm-kubernetes.git']])
            }
        }
```


2.  Build our application using Maven into a JAR file in our target directory
```
        stage('Build Artifact') {
            steps {
               sh 'mvn clean install'
            }
        }
```

3. Perform a Unit Test
```
               stage('Unit Test') {
            steps {
               sh 'mvn test'
            }
        }
```
4. We will build our docker Image and tag it with the $BUILD_NUMBER
We will store the docker build in a variable dockerImage, to allow easy tagging with the build number before pushing to the repository.

```
        stage('Build Docker Image') {
            steps {
               script{
                   dockerImage = docker.build registry
                   dockerImage.tag("$BUILD_NUMBER")
               }
            }
        }
```
5. Push the Docker Image to our ECR registry
The push command is available in our ECR Registry
Select your repo and click the view push commands button.
![get push command](./images/ecr1.png)
Retrive an identification token and authenticate the docker client by running the AWSCLI command below
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 185439933271.dkr.ecr.us-east-1.amazonaws.com
![get ecr token](./images/ecr2.png)
```
        stage('Push Docker Image') {
            steps {
               script{
                   sh'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 185439933271.dkr.ecr.us-east-1.amazonaws.com'
                   sh'docker push 185439933271.dkr.ecr.us-east-1.amazonaws.com/my-ecr-repo:$BUILD_NUMBER'
               }
            }
        }
```

6. Deploy our comtainer into our EKS cluster using a helm chart

```
        stage('Helm Deployment') {
            steps {
               script{
                   sh"helm upgrade first --install mychart --namespace helm-deployment --set image.tag=$BUILD_NUMBER"
                  
               }
            }
        }
```
```
pipeline {
    
    tools {
        maven 'Maven3'
    }
    agent any
    
    environment {
        registry = "185439933271.dkr.ecr.us-east-1.amazonaws.com/my-ecr-repo"
    }
    stages {
        stage('Git Checkout') {
            steps {
               checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/deleonab/cicd-jenkins-docker-helm-kubernetes.git']])
            }
        }
        stage('Build Artifact') {
            steps {
               sh 'mvn clean install'
            }
        }
         stage('Unit Test') {
            steps {
               sh 'mvn test'
            }
        }
        stage('Build Docker Image') {
            steps {
               script{
                   dockerImage = docker.build registry
                   dockerImage.tag("$BUILD_NUMBER")
               }
            }
        }
        stage('Push Docker Image') {
            steps {
               script{
                   sh'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 185439933271.dkr.ecr.us-east-1.amazonaws.com'
                   sh'docker push 185439933271.dkr.ecr.us-east-1.amazonaws.com/my-ecr-repo:$BUILD_NUMBER'
               }
            }
        }
        stage('Helm Deployment') {
            steps {
               script{
                   sh"helm upgrade first --install mychart --namespace helm-deployment --set image.tag=$BUILD_NUMBER"
                  
               }
            }
        }
    } 
}

```


Let's run our pipeline from inside Jenkins
Let's go to our project and click 'Build Now'

![build Now](./images/pipeline1.png)

After a few hiccups, Our pipeline ran successfully

![build Now](./images/pipeline2.png)


We can view our deployed application by using the LoadBalancer url which exposes it to the internet
We can get this by running kubectl get svc.

Let's view all our resources in our namespace helm-deployment
Let's view our 2 worker nodes
```
kubectl get nodes

jenkins@ip-172-31-53-176:~$ kubectl get nodes
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-23-179.ec2.internal   Ready    <none>   3d    v1.25.9-eks-0a21954
ip-192-168-40-4.ec2.internal     Ready    <none>   3d    v1.25.9-eks-0a21954
```
Let's see our deployed application running in a pod
```
kubectl get pods -n helm-deployment

jenkins@ip-172-31-53-176:~$ kubectl get pods -n helm-deployment
NAME                             READY   STATUS    RESTARTS   AGE
first-mychart-64b4fd4498-89hbp   1/1     Running   0          3m32s

```

Let's view our LoadBalancer service which exposes our application to the world

```
kubectl get svc -n helm-deployment
```
The external IP is our LoadBalancer url to view our application in a browser
```
jenkins@ip-172-31-53-176:~$ kubectl get svc -n helm-deployment
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)        AGE
first-mychart   LoadBalancer   10.100.118.220   ad94ffdc78810478780c6a533fb2eb6a-2046723893.us-east-1.elb.amazonaws.com   80:30155/TCP   2d
```

Let's go to this url in our browser ad94ffdc78810478780c6a533fb2eb6a-2046723893.us-east-1.elb.amazonaws.com


![final app](./images/final.png)

# 🚀 I'm are always open to your feedback.  Please contact as bellow information:
### [Contact ]
* [Name: nho Luong]
* [Skype](luongutnho_skype)
* [Github](https://github.com/nholuongut/)
* [Linkedin](https://www.linkedin.com/in/nholuong/)
* [Email Address](luongutnho@hotmail.com)
* [PayPal.me](https://www.paypal.com/paypalme/nholuongut)

![](https://i.imgur.com/waxVImv.png)
![](Donate.png)
[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/nholuong)

# License
* Nho Luong (c). All Rights Reserved.🌟













