#!/usr/bin/env groovy

library identifier: "jenkins-shared-library@master", retriever: modernSCM([
    $class: "GitSCMSource",
    remote: "git@github.com:TheAbys/devops-bootcamp-08-jenkins-shared-library.git",
    credentialsId: "capstone-project-1"
])

pipeline {   
  agent any
  tools {
    maven 'Maven'
  }

  environment {
    IMAGE_NAME = '904233123058.dkr.ecr.eu-central-1.amazonaws.com:java-maven-2.0'
  }
  stages {
    stage("build app") {
      steps {
        script {
          echo 'building application jar...'
          buildJar()
        }
      }
    }
    stage("build image") {
      steps {
        script {
          echo 'building docker image...'
          buildImage(env.IMAGE_NAME)
          dockerLogin()
          dockerPush(env.IMAGE_NAME)
        }
      }
    }
    stage("provision server") {
      environment {
        AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
        TF_VAR_env_prefix = 'test'
      }
      steps {
        script {
          dir('terraform') {
            sh "terraform init"
            sh "terraform apply --auto-approve"
            EC2_PUBLIC_IP = sh(
              script: "terraform output ec2-public_ip",
              returnStdout: true
            ).trim()
          }
        }
      }
    }
    stage("deploy") {
      environment {
        DOCKER_CREDS = credentials('capstone-project-1-ecr-credentials')
      }
      steps {
        script {
          echo "waiting for EC2 server to initialize"
          sleep(time: 90, unit: "SECONDS")

          echo 'deploying docker image to EC2...'
          echo "${EC2_PUBLIC_IP}"
          
          def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
          def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"

          sshagent(['server-ssh-key']) {
            sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
            sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
            sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
          }
        }
      }
    }               
  }
}
