pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_DEFAULT_REGION    = 'us-east-2'
    }

    parameters {
        choice(name: 'ACTION', choices: ['apply','destroy'], description: 'Deploy or Destroy infrastructure')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/chandu245/nginx-terraform-ansible-jenkins.git'
            }
        }

        stage('Terraform') {
            steps {
                dir('terraform') {
                    bat 'terraform init'
                    script {
                        if (params.ACTION == 'apply') {
                            bat 'terraform apply -auto-approve'
                        } else {
                            bat 'terraform destroy -auto-approve'
                        }
                    }
                }
            }
        }

        stage('Update Ansible Inventory') {
            when { expression { params.ACTION == 'apply' } }
            steps {
                script {
                    dir('terraform') {
                        def ip = bat(script: 'terraform output -raw public_ip', returnStdout: true).trim()
                        env.EC2_IP = ip
                    }
                    powershell """
                    (Get-Content ansible\\inventory.ini) -replace 'PLACEHOLDER', '$env:EC2_IP' | Set-Content ansible\\inventory.ini
                    """
                }
            }
        }

        stage('Run Ansible') {
            when { expression { params.ACTION == 'apply' } }
            steps {
                bat 'ansible-playbook -i ansible\\inventory.ini ansible\\playbook.yml'
            }
        }

        stage('Website URL') {
            when { expression { params.ACTION == 'apply' } }
            steps {
                echo "Website available at: http://${env.EC2_IP}"
            }
        }
    }

    post {
        success {
            echo "Pipeline finished successfully."
        }
        failure {
            echo "Pipeline failed. Check logs."
        }
    }
}