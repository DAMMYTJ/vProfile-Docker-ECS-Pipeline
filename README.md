# vProfile Docker ECS Pipeline

## Overview

This project demonstrates a CI/CD pipeline for deploying a front-end application to AWS ECS (Elastic Container Service) using Docker and Jenkins. The pipeline includes stages for fetching the code, running tests, performing code analysis, building Docker images, pushing the images to Amazon ECR, and deploying the application to ECS.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Setup Instructions](#setup-instructions)
  - [1. AWS Configuration](#1-aws-configuration)
  - [2. Jenkins Setup](#2-jenkins-setup)
- [Pipeline Details](#pipeline-details)
- [How to Run](#how-to-run)

## Prerequisites

Before starting, ensure you have the following:

1. **AWS Account**
2. **AWS CLI** installed and configured
3. **Docker** installed
4. **Jenkins** set up with Docker and AWS plugins
5. **GitHub** repository for the application code

## Architecture

The pipeline performs the following steps:

1. **Fetch Code**: Retrieves the latest code from the GitHub repository.
2. **Test**: Runs unit tests using Maven.
3. **Code Analysis**: Performs code analysis with Checkstyle and SonarQube.
4. **Build App Image**: Builds a Docker image of the application.
5. **Upload App Image**: Pushes the Docker image to Amazon ECR.
6. **Deploy to ECS**: Updates the ECS service to deploy the new Docker image.

## Setup Instructions

### 1. AWS Configuration

#### Create a Target Group

1. **Navigate to EC2**: Go to the AWS Management Console and navigate to the EC2 service.
2. **Create Target Group**:
   - Choose `Target Groups` from the left-hand menu.
   - Click `Create target group`.
   - Choose `Instances` as the target type.
   - Name the target group `vpro ECS TG`.
   - Set the protocol to `HTTP` and the port to `80`.
   - For the health check path, set it to `/login`.
   - Click `Next`, then `Create`.

#### Configure ECS Service

1. **Create ECS Cluster**:
   - Navigate to the ECS service in the AWS Management Console.
   - Click `Clusters`, then `Create cluster`.
   - Choose `Networking only (Fargate)`, and click `Next`.
   - Name the cluster `vprofile`.

2. **Create Task Definition**:
   - Go to `Task Definitions` and click `Create new Task Definition`.
   - Choose `Fargate`.
   - Set the task definition name (e.g., `vprofile-task`).
   - Add a container, configure the image, port mappings, and memory settings.
   - Click `Create`.

3. **Create Service**:
   - Go to the `Services` tab within your cluster.
   - Click `Create`, and configure the service settings.
   - Name the service `vprofileappsvc`.
   - Set the number of tasks to `1`.
   - Attach the load balancer and select the target group `vpro ECS TG`.
   - Click `Create`.

### 2. Jenkins Setup

#### Install Plugins

1. **Navigate to Jenkins**:
   - Go to `Manage Jenkins` > `Manage Plugins`.
2. **Install Plugins**:
   - Install `AWS Steps Plugin` and `Pipeline: AWS Steps`.

#### Create Pipeline Job

1. **Create a New Pipeline Job**:
   - Go to `New Item`, enter a name (`CICD Pipeline ECS`), and select `Pipeline`.
   - Click `OK`.
2. **Configure Pipeline Script**:
   - In the pipeline configuration, use the following script:

```groovy
pipeline {
    agent any
    tools {
        maven "MAVEN3"
        jdk "OracleJDK8"
    }

    environment {
        registryCredential = 'ecr:us-east-2:awscreds'
        appRegistry = "951401132355.dkr.ecr.us-east-2.amazonaws.com/vprofileappimg"
        vprofileRegistry = "https://951401132355.dkr.ecr.us-east-2.amazonaws.com"
        cluster = "vprofile"
        service = "vprofileappsvc"
    }

    stages {
        stage('Fetch code') {
            steps {
                git branch: 'docker', url: 'https://github.com/DAMMYTJ/javaapprepo.git'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('build && SonarQube analysis') {
            environment {
                scannerHome = tool 'sonar4.7'
            }
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                       -Dsonar.projectName=vprofile-repo \
                       -Dsonar.projectVersion=1.0 \
                       -Dsonar.sources=src/ \
                       -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                       -Dsonar.junit.reportsPath=target/surefire-reports/ \
                       -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                       -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build(appRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                }
            }
        }

        stage('Upload App Image') {
            steps {
                script {
                    docker.withRegistry(vprofileRegistry, registryCredential) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy to ecs') {
            steps {
                withAWS(credentials: 'awscreds', region: 'us-east-2') {
                    sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
                }
            }
        }
    }
}
```

## Pipeline Details

The Jenkins pipeline is defined in the `Jenkinsfile` and consists of the following stages:

- **Fetch Code**: Clones the GitHub repository.
- **Test**: Runs the tests using Maven.
- **Code Analysis**: Analyzes the code with Checkstyle and SonarQube.
- **Build App Image**: Builds the Docker image for the application.
- **Upload App Image**: Pushes the Docker image to Amazon ECR.
- **Deploy to ECS**: Deploys the Docker image to the ECS service.

## How to Run

1. **Set Up AWS Credentials**: Ensure that the AWS CLI is configured with the necessary credentials and permissions to access ECS and ECR.
2. **Configure Jenkins Credentials**: Set up Jenkins credentials for AWS and Docker registry.
3. **Run the Pipeline**: Start the Jenkins pipeline job.
4. **Monitor the Pipeline**: Check the job's progress and logs in Jenkins.
5. **Verify Deployment**: After the pipeline completes, verify that the application is running by accessing the load balancer's URL.


<img w<img width="1245" alt="Screenshot 2024-05-20 at 10 00 57 PM" src="https://github.com/DAMMYTJ/vProfile-Docker-ECS-Pipeline/assets/111458165/50f42c82-591f-47f3-a756-0ef306f22866">
<img width="1245" alt="Screenshot 2024-05-20 at 10 00 06 PM" src="https://github.com/DAMMYTJ/vProfile-Docker-ECS-Pipeline/assets/111458165/a1652aa2-4107-47ca-bbcf-0622acca46ce">





<img width="1444" alt="Screenshot 2024-05-20 at 11 42 20 PM" src="https://github.com/DAMMYTJ/vProfile-Docker-ECS-Pipeline/assets/111458165/d0d6595e-03d6-42ff-aeb8-d00c573bdb9e"><img width="1446" alt="Screenshot 2024-05-20 at 11 43 09 PM" src="https://github.com/DAMMYTJ/vProfile-Docker-ECS-Pipeline/assets/111458165/60cf81ca-33e4-4f20-aa95-572feed4b7a1">


















