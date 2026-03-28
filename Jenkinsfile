pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-1'
        TF_IN_AUTOMATION = 'true'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/chandu245/nginx-terraform-ansible-jenkins.git'
            }
        }

        stage('Verify AWS IAM Role') {
            steps {
                sh 'aws sts get-caller-identity'
            }
        }

        stage('Terraform Init') {
            steps {
                dir('terraform') {
                    sh 'terraform init'
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                dir('terraform') {
                    sh 'terraform apply -auto-approve -var="key_name=server"'
                    script {
                        env.EC2_IP = sh(script: 'terraform output -raw ec2_public_ip', returnStdout: true).trim()
                    }
                }
            }
        }

        stage('Wait for EC2 SSH') {
            steps {
                sh '''
                    echo "Waiting for EC2 SSH to become available..."
                    sleep 60
                '''
            }
        }

        stage('Create Ansible Inventory') {
            steps {
                dir('ansible') {
                    writeFile file: 'inventory.ini', text: """[webserver]
${env.EC2_IP} ansible_user=ec2-user ansible_python_interpreter=/usr/bin/python3
"""
                    sh 'cat inventory.ini'
                }
            }
        }

        stage('Ansible Deploy Nginx') {
            steps {
                dir('ansible') {
                    sshagent(credentials: ['ec2-ssh-key']) {
                        sh '''
                            export ANSIBLE_HOST_KEY_CHECKING=False
                            ansible-playbook -i inventory.ini nginx.yml
                        '''
                    }
                }
            }
        }

        stage('Show Website URL') {
            steps {
                sh 'echo "Website URL: http://${EC2_IP}"'
            }
        }
    }

    post {
        success {
            echo "Deployment successful! Open: http://${env.EC2_IP}"
        }
        failure {
            echo "Deployment failed. Check console output."
        }
    }
}
