## Overview

Welcome to our comprehensive CI/CD pipeline repository! Here, we provide a complete guide to setting up and understanding every stage of the CI/CD process. From integrating SonarQube for code quality analysis, Nexus for artifact management, and Maven for builds, to exploring other essential plugins, our repository aims to equip with practical knowledge.
By the end of this journey, we'll not only master application-level CI/CD setups but also delve into system-level monitoring using Prometheus and Grafana. We will be deploying Board Game Database Full-Stack Web Application using CI/CD Pipeline and finally set up monitoring for our Application and for jenkins.

## Phases
Phase 1 - Setting up Infrastructure

Phase 2 - Creating git repository and pushing the source code

Phase 3 - CI/CD Pipeline, Deployemnt of Application, Mail notification

Phase 4 - Setup monitoring

## Setting up Infrastructure

### Create VPC
AWS management console --> VPC --> create VPC

In AWS Console, create a VPC in your prefered location. The default VPC can also be used.

### Create security group 
AWS management console --> EC2 --> security groups

Create a security group "ci/cd-sg" and add inbound rules to open the required ports as shown in the below image. These are the ports that we would be using in this project.

![inbound rules](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/security-group.png?raw=true)

### Create virtual machines (EC2 Instances)

Create 3 new instances for kubernetes cluster of Ubuntu 22.04 AMI and instance type t2.medium. Add the security group configured in the above step (ci/cd-sg). Configure storage as 25 Gib. Launch Instances. These can be named as master, slave-1 and slave-2.

### Setup Kubernetes on each node.

Connect to the instances through EC2 Instance Connect. Run the below commands in the master, slave-1 and slave-2 nodes.
```bash
$ sudo apt-get update
$ sudo apt install docker.io -y
$ sudo chmod 666 /var/run/docker.sock
$ sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
$ sudo mkdir -p -m 755 /etc/apt/keyrings
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo apt update
$ sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
```

Run the below commands only on master node
```bash
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16

The output gives a command using which the worker nodes can be joined to the master node. Run that command on both the worker nodes(slave-1 and slave-2)
The command should be something like this

$ kubeadm join 172.31.39.68:6443 --token qeno4a.aramaai6vbw2fn3f \
        --discovery-token-ca-cert-hash sha256:2c05796b107d2bbd0e5eed6e69343bc48fa3023aa24e260aca49efecd51f481d
```

On master node,
```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
```

kubernetes is setup. 

### Create VM's for Sonar, Nexus and Jenkins

Launch two instances for sonar and nexus.
parameters : Ubuntu 22.04 AMI and instance type t2.medium. Add the security group 'ci/cd-sg'. Configure storage as 20 Gib.
Launch a bigger instance for Jenkins of type Ubuntu 22.04 t2.large with storage of 30 Gib. Add the existing sg 'ci/cd-sg'
Let's setup each of them.

#### Configure SonarQube

In SonarQube Instance,
```bash
Add Docker's official GPG key:
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl
$ sudo install -m 0755 -d /etc/apt/keyrings
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc

Add the repository to Apt sources:
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
$ sudo chmod 666 /var/run/docker.sock
$ docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
$ docker ps
```

Hit Ipaddress:9000 in your browser. It should give you the login page for Sonar.
Default user and password is admin and admin respectively. Update your password and sonar is ready to use.





Configuring Nexus

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo chmod 666 /var/run/docker.sock

creating docker container for nexus --> docker run -d --name Nexus -p 8081:8081 sonatype/nexus3
docker ps
Hit ipaddress:8081. Nexus should be up and running. 
Sign in --> default username is admin. To get the password, follow the below steps
docker exec -it b95917a0a690 /bin/bash
cat /opt/sonatype/sonatype-work/nexus3/admin.password - get the password from here and login to nexus after which you can update your password. (Janvi03@nexus)

Configuring Jenkins

Connect to the instance
sudo apt update
sudo apt install openjdk-17-jre-headless -y
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins

jenkins is installed.
Install docker on jenkins instance
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo chmod 666 /var/run/docker.sock

install trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy


Hit Ipaddress:8080 --> get the passowrd from 
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Install suggested plugins
create your user and password - save and finish














