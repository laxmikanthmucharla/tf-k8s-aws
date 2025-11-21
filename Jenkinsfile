#!/usr/bin/env groovy
pipeline {
    agent any
    parameters {
        choice(
            name: 'ACTION',
            choices: ['apply', 'destroy'],
            description: 'Choose whether to apply or destroy the Terraform infrastructure'
        )
    }
    environment {
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_DEFAULT_REGION = "us-east-1"
		DOCKER_IMAGE = 'myapp'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
    }
    stages {
        stage("Terraform Action") {
            steps {
                script {
                    dir('eks-cluster') {
                        sh "terraform init"
                        if (params.ACTION == 'apply') {
                            echo "Applying Terraform configuration..."
                            sh "terraform apply -auto-approve"
                        } else if (params.ACTION == 'destroy') {
                            echo "Destroying Terraform infrastructure..."
                            sh "terraform destroy -auto-approve"
                        }
                    }
                }
            }
        }
		stage('Terraform Apply') {
   steps {
       dir('eks-cluster') {
           script {
               sh 'terraform init'
               echo "Applying Terraform configuration..."
               sh """
                   terraform apply -auto-approve \
                   -var="vpc_cidr_block=10.0.0.0/16" \
                   -var='private_subnet_cidr_blocks=["10.0.3.0/24","10.0.4.0/24"]' \
                   -var='public_subnet_cidr_blocks=["10.0.1.0/24","10.0.2.0/24"]'
               """
           }
       }
   }
}
        stage("Deploy to EKS") {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                script {
                    dir('Kubernetes') {
                        sh "aws eks update-kubeconfig --name eks-cluster --region us-east-1"
                        sh "cat /var/lib/jenkins/.kube/config"
                        sh "kubectl apply -f nginx-deployment.yaml -n default"
                        sh "kubectl apply -f nginx-service.yaml -n default"
                    }
                }
            }
        }
        stage("Cleanup Kubernetes Resources") {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                script {
                    dir('Kubernetes') {
                        // Update kubeconfig first (in case cluster still exists)
                        sh '''
                            if aws eks describe-cluster --name eks-cluster --region us-east-1 >/dev/null 2>&1; then
                                echo "EKS cluster exists, updating kubeconfig..."
                                aws eks update-kubeconfig --name eks-cluster --region us-east-1
                                echo "Cleaning up Kubernetes resources..."
                                kubectl delete -f nginx-service.yaml -n default --ignore-not-found=true
                                kubectl delete -f nginx-deployment.yaml -n default --ignore-not-found=true
                            else
                                echo "EKS cluster does not exist, skipping Kubernetes cleanup"
                            fi
                        '''
                    }
                }
            }
        }
    }
    post {
        always {
            echo "Pipeline completed with action: ${params.ACTION}"
        }
        success {
            script {
                if (params.ACTION == 'apply') {
                    echo "Infrastructure successfully created and applications deployed!"
                } else {
                    echo "Infrastructure successfully destroyed!"
                }
            }
        }
        failure {
            script {
                if (params.ACTION == 'apply') {
                    echo "Failed to create infrastructure or deploy applications"
                } else {
                    echo "Failed to destroy infrastructure"
                }
            }
        }
    }
}
