#!/usr/bin/groovy

pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    environment {
        IMAGE_NAME = 'khaledmohamed98/my-app:java-maven-2.0'
    }
    stages {
        stage('build app') {
            steps {
                script {
                    echo 'building application jar...'
                    sh 'mvn package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo 'building the docker image...'
                    sh "docker build -t $IMAGE_NAME ."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-private-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh "echo $PASS | docker login -u $USER --password-stdin"
                    sh "docker push $IMAGE_NAME"
                    }
                }
            }
        }
        stage('provision server') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                TF_VAR_env_prefix = 'test'
            }
            steps {
                script {
                    echo 'provioning server .....'
                    dir('terraform') {
                        sh 'terraform init -reconfigure'
                        sh 'terraform apply --auto-approve'
                        EC2_PUBLIC_IP = sh(
                            script: 'terraform output ec2_public_ip',
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }
        stage('deploy') {
            environment {
                DOCKER_CREDS = credentials('docker-hub-private-repo')
            }
            steps {
                script {
                    echo 'deploying app .... '
                    echo 'waiting for EC2 server to initialize' 
                    sleep(time: 90, unit: 'SECONDS') 

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

