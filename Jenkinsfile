pipeline {
    agent any
    environment {
        //be sure to replace "bhavukm" with your own Docker Hub username
        DOCKER_IMAGE_REPO = "docshiva"
        DOCKER_IMAGE_NAME = "train-schedule"
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build("$DOCKER_IMAGE_REPO/$DOCKER_IMAGE_NAME")
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = "1"
            }
            steps {
                script {
                    kubeconfig(credentialsId: 'kubeconfig') {
                        sh 'sed -i "s/CANARY_REPLICAS/$CANARY_REPLICAS/g" train-schedule-kube-canary.yml'
                        sh 'sed -i "s/DOCKER_IMAGE_REPO/$DOCKER_IMAGE_REPO/g" train-schedule-kube-canary.yml'
                        sh 'sed -i "s/DOCKER_IMAGE_NAME/$DOCKER_IMAGE_NAME/g" train-schedule-kube-canary.yml'
                        sh 'sed -i "s/BUILD_NUMBER/$BUILD_NUMBER/g" train-schedule-kube-canary.yml'
                        sh 'echo `kubectl delete ns canary`'
                        sh 'echo `kubectl create ns canary`'
                        sh 'kubectl apply -f train-schedule-kube-canary.yml -n canary'
                        sh '''#!/bin/bash
                        count=10
                        while [ $count -ne "0" ];
                        do
                          if [ "$(kubectl get po -l app=train-schedule -n canary | grep '1/1') | wc -l" == "1" ]; then
                            echo "Successfully Deployed"
                            exit 0
                          else
                            sleep 5
                            count=$((count - 1))
                          fi
                        done
                        echo "Deployment Failed"
                        exit 1
                        '''
                        //kubernetes ( yamlFile: 'train-schedule-kube-canary.yml')
                        //kubernetesDeploy(configs: "train-schedule-kube-canary.yaml")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = "0"
            }
            steps {
                script {
                    input 'Deploy to Production?'
                    milestone(1)
                    kubeconfig(credentialsId: 'kubeconfig') {
                        sh 'sed -i "s/DOCKER_IMAGE_REPO/$DOCKER_IMAGE_REPO/g" train-schedule-kube.yml'
                        sh 'sed -i "s/DOCKER_IMAGE_NAME/$DOCKER_IMAGE_NAME/g" train-schedule-kube.yml'
                        sh 'sed -i "s/BUILD_NUMBER/$BUILD_NUMBER/g" train-schedule-kube.yml'
                        sh 'echo `kubectl delete ns production`'
                        sh 'echo `kubectl create ns production`'
                        //sh 'kubernetes apply -f train-schedule-kube-canary.yml -n production'
                        sh 'kubernetes apply -f train-schedule-kube.yml -n production'
                        sh '''#!/bin/bash
                        count=10
                        while [ $count -gt "0" ];
                        do
                          if [ "$(kubectl get po -l app=train-schedule -n production | grep '1/1') | wc -l" == "2" ]; then
                            echo "Successfully Deployed"
                            exit 0
                          else
                            sleep 5
                            count=$((count - 1))
                          fi
                        done
                        echo "Deployment Failed"
                        exit 1
                        '''
                    }
                    //kubernetes ( yamlFile: 'train-schedule-kube-canary.yml')
                    //kubernetes ( yamlFile: 'train-schedule-kube.yml')
                }
            }
        }
    }
}
