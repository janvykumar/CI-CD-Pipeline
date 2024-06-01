## Overview

Welcome to our comprehensive CI/CD pipeline repository! Here, we provide a complete guide to setting up and understanding every stage of the CI/CD process. From integrating SonarQube for code quality analysis, Nexus for artifact management, and Maven for builds, to exploring other essential plugins, our repository aims to equip with practical knowledge.
By the end of this journey, we'll not only master application-level CI/CD setups but also delve into system-level monitoring using Prometheus and Grafana. We will be deploying Board Game Database Full-Stack Web Application using CI/CD Pipeline and finally set up monitoring for our Application and for jenkins.

## Phases
Phase 1 - Setting up Infrastructure

Phase 2 - Creating git repository and pushing the source code

Phase 3 - CI/CD Pipeline, Deployemnt of Application, Mail notification

Phase 4 - Setup monitoring

## Phase 1 : Setting up Infrastructure

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

![K8 setup](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/K8%20setup.png?raw=true)

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

![SonarQube](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/SonarQube.png?raw=true)

#### Configure Nexus

In Nexus instance,

Run the same commands to install Docker as done while configuring SonarQube. Then create docker container by running the below command.
```bash
$ docker run -d --name Nexus -p 8081:8081 sonatype/nexus3
$ docker ps
```
Hit ipaddress:8081. Nexus should be up and running. 
Sign in --> default username is admin. To get the password, follow the below steps.
```bash
$ docker exec -it b95917a0a690 /bin/bash
$ cat /opt/sonatype/sonatype-work/nexus3/admin.password
```
Get the password from the above path and login to nexus after which you can update your password.

![Nexus](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/nexus.png?raw=true)

#### Configure Jenkins

Connect to the Jenkins instance,
Install Java
```bash
$ sudo apt update
$ sudo apt install openjdk-17-jre-headless -y
```
Install Jenkins
```bash
$ sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
$ echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install jenkins
```

Install Docker on jenkins instance. Again follow the same commands as done while configuring Sonar and Nexus.

Install trivy
```bash
$ sudo apt-get install wget apt-transport-https gnupg lsb-release
$ wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
$ echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
$ sudo apt-get update
$ sudo apt-get install trivy
```

Install kubectl
```bash
$ curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin
$ kubectl version --short --client
```

Hit Ipaddress:8080 --> get the passowrd from:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Install suggested plugins
Create your username and password - save and finish.

![Jenkins](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/Jenkins.png?raw=true)

## Phase 2 : Creating git repository and pushing the source code

Create your Git Repository. 
```bash
$ git clone <your git repo url>
$ git checkout <your branch>
Add your source code.
$ git add .
$ git commit -m "your commit message"
$ git push
```

To clone this repo,
```bash
$ git clone https://github.com/janvykumar/CI-CD-Pipeline.git
```

## Phase 3 : CI/CD Pipeline, Deployemnt of Application, Mail notification

### Install Plugins for Jenkins

Jenkins Dashboard --> Manage jenkins --> plugins --> Install Eclipse Temurin installer, Config File Provider(for maven), Pipeline Maven Integration, Maven Integration, SonarQube Scanner, Docker, Docker Pipeline, Kubernetes, Kubernetes Client API, Kubernetes Credentials and Kubernetes CLI.

### Configure the plugins

Manage jenkins --> tools -->

Add jdk - jdk17 - check mark 'install automatically' option --> Install from adoptium.net --> select jdk-17.0.9+9

Add sonarQube scanner --> name can be given as 'sonar-scanner' --> go with the needed version(I am going with the latest)

Add maven --> name - 'maven3' --> version 3.6.1

Add docker --> name - 'docker' --> download from docker.io --> latest version

Save

### Configure sonarQube server in Jenkins

Manage jenkins --> Credentials --> Global --> Add Credentials --> kind is 'secret text' --> Now, we need the tokens.

To get the sonar tokens, 

Go to SonarQube UI --> administration --> security --> users --> click on update token --> give name as 'sonar-token' --> Generate. Copy the token.

Come back to jenkins and add credentials --> give the token in 'secret' --> id is 'sonar-token' --> description is 'sonar-token' --> Save. 

Now, Let's configure SonarQube in Jenkins,

Go to Manage jenkins --> system --> Add sonarQube --> Name is 'sonar' --> url is 'http://IPaddress_of_SonarInstance:9000/' --> server authentication token is 'sonar-token' --> Save.

### Add GitHub credentials in Jenkins

Manage jenkins --> credentials --> system --> global credentials --> add credentials -> Add username and personal access token of your GitHub Account.

### Create SonarQube webhook

Go to Sonar UI --> administration --> configuration --> webhooks --> create --> name - 'jenkins' --> url should be '<jenkins-url>/sonarqube-webhook/' --> Create.

![sonar-webhook](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/SonarQube-webhook.png?raw=true)

### Add maven components to pom.xml file

In pom.xml file, you will see "distributionManagement" section where "maven-releases" and "maven-snapshots" are defined. We need to get the url's for both from Nexus and put it in pom.xml file. Refer pom.xml file of this repo.

To get the url, 
Go to Nexus UI --> Browse --> click on 'copy' to the right of maven-releases --> Copy the url --> put it in pom.xml inside the "maven-releases" section.
Similarly, get the url of "maven-snapshots" and add it in pom.xml.

```bash
<distributionManagement>
<repository>
    <id>maven-releases</id>
    <url>http://ip-address:8081/repository/maven-releases/</url>
</repository>
<snapshotRepository>
    <id>maven-snapshots</id>
    <url>http://ip-address:8081/repository/maven-snapshots/</url>
</snapshotRepository>	
</distributionManagement>
```

### Add maven configs in Jenkins

Jenkins --> Manage jenkins --> managed files --> Global Maven settings.xml --> id is 'global-settings' --> save --> Add the maven configs in the settings.xml file as below --> Submit.

```bash
    <server>
      <id>maven-releases</id>
      <username>your nexus username</username>
      <password>youe nexus password</password>
    </server>
    
    <server>
      <id>maven-snapshots</id>
      <username>your nexus username</username>
      <password>your nexus password</password>
    </server>
```

### Deploy Application to kubernetes

#### Create service account and rules for jenkins. 

In master node, create a namespace called 'webapps' and add svc.yaml, role.yaml, bind.yaml and sec.yaml and apply them.

svc.yaml
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

role.yaml
```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

bind.yaml
```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins
```

sec.yaml
```bash
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: jenkins
```

Create namespace and apply the above yaml files.
```bash
$ kubectl create ns webapps
$ kubectl apply -f svc.yaml
$ kubectl apply -f role.yaml
$ kubectl apply -f bind.yaml
$ kubectl apply -f sec.yaml -n webapps
```

Get the token to authenticate Kubernetes to Jenkins.
```bash
$ kubectl describe secret mysecretname -n webapps
```

Go to Jenkins --> Add a secret text global credential --> paste the token inside secret --> id is 'k8-cred' --> Save.

Add deployemnt-service.yaml file in your repo. Refer "deployment-service.yaml" file of this repo.

### Configure Mail Notification
Go to your Google Account --> manage google account --> security --> 2 step verification --> app passowrd --> give 'jenkins' as name and you will get a token

Jenkins --> manage jenkins --> system --> configure extended mail notification and email notification. Make sure to add global credential for mail.

Below is a screenshot of all the credentials that needs to be added in Jenkins

![Creds](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/Jenkins-cred.png?raw=true)

### Create pipeline

Jenkins Dashboard --> create new pipeline --> give name as 'BoardGame' --> choose 'pipeline' --> let's configure our pipeline.

Inside Configure, write the pipeline script. Refer to 'pipeline-script' file of this repo to get the script.

### Run the job.

![successful-job](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/Build.png?raw=true)

![SonarQube](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/Sonar-boardgame.png?raw=true)

![nexus](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/maven-boardgame.png?raw=true)

![mail](https://github.com/janvykumar/CI-CD-Pipeline/blob/main/mail-notification.png?raw=true)

With that, our CI/CD Pipeline in complete

## Phase 4 - Setup monitoring

Create an Instance (Ubuntu 22.04) named "monitoring" of t2.medium type, having "ci/cd-sg" security group. Connect to the instance and follow as below.

### Install prometheus
```bash
$ wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
$ tar -xvf prometheus-2.52.0.linux-amd64.tar.gz
$ cd prometheus-2.52.0.linux-amd64/
$ ./prometheus &
```
Hit http://IP:9090/ in your browser.

### Install grafana
```bash
$ sudo apt-get install -y adduser libfontconfig1 musl
$ wget https://dl.grafana.com/enterprise/release/grafana-enterprise_11.0.0_amd64.deb
$ sudo dpkg -i grafana-enterprise_11.0.0_amd64.deb
$ sudo /bin/systemctl start grafana-server
```
Hit http://IP:3000/
Default username and password is admin and admin. Update it.

### Install blackbox exporter
```bash
$ wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
$ tar -xvf blackbox_exporter-0.25.0.linux-amd64.tar.gz
$ cd blackbox_exporter-0.25.0.linux-amd64/
$ ./blackbox_exporter &
```
hit http:IP:9115/

### Configure prometheus.yaml file
```bash
cd prometheus-2.52.0.linux-amd64/
```

Add the below section in your prometheus.yml (Make sure to provide the IP address of your instance) --> Save.
```bash
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]     

  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - http://prometheus.io    # Target to probe with http.
        - http://65.0.129.208:30685 # Target to probe with http on port 8080.
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 13.201.34.240:9115  # The blackbox exporter's real hostname:port.
```

###  Restart prometheus
```bash
$ pgrep prometheus
$ kill <id>
$ ./prometheus &
```
Refresh prometheus UI --> status --> targets --> You should be able to see the targets up and healthy.

![prometheus-targets]()

### Add prometheus as data source inside Grafana

Go to grafana --> connections --> data sources --> add data source --> prometheus --> provide prometheus url --> save and test.

+ sign --> import dashboard

Go to https://grafana.com/grafana/dashboards/7587-prometheus-blackbox-exporter/ to export the dashboard to our grafana dashboard --> get the id from the link.

Come back to your grafana UI for importing dashboard --> paste and load the dashboard --> select prometheus for signcl-prometheus --> import 

You will see all the metrices. 

![Application-monitoring]()

We have setup monitoring for the Application!

### System monitoring

#### let's monitor Jenkins 

Go to Jenkins --> manage jenkins --> plugins --> Install "Prometheus metrics" plugin.

In jenkins instance,
```bash
$ wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
$ tar -xvf node_exporter-1.8.1.linux-amd64.tar.gz
$ cd node_exporter-1.8.1.linux-amd64/
$ ./node_exporter &
```
Hit http://IP:9100/

In monitoring instance,
```bash
$ cd prometheus-2.52.0.linux-amd64
```

Edit prometheus.yml file, add the below section (Edit as per your instance IP) --> Save.
```bash
- job_name: 'node_exporter'
    static_configs:
      - targets: ['13.235.95.88:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['13.235.95.88:8080']
```

Restart prometheus.
```bash
```bash
$ pgrep prometheus
$ kill <id>
$ ./prometheus &
```
You will see jenkins added to prometheus. Check out the above screenshot of prometheus targets.

#### Import dashboard

Import https://grafana.com/grafana/dashboards/1860-node-exporter-full/ to our Grafana dashboard --> get the ID from the link.

Copy ID in our dashboard and load --> select prometheus datasource.

Our system monitoring for Jenkins is ready.

![system-monitoring]()















