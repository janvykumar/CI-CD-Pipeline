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
