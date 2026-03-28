pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-2'
        TF_IN_AUTOMATION = 'true'
    }

    stages {
        stage('Verify AWS IAM Role') {
            steps {
                sh '''
                    echo "Checking AWS access using IAM Role..."
                    aws sts get-caller-identity
                '''
            }
        }

        stage('Terraform Init') {
            steps {
                dir('terraform') {
                    sh '''
                        echo "Initializing Terraform..."
                        terraform init
                    '''
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                dir('terraform') {
                    sh '''
                        echo "Creating EC2 using Terraform..."
                        terraform apply -auto-approve -var="key_name=server"
                    '''
                    script {
                        env.EC2_IP = sh(
                            script: 'terraform output -raw ec2_public_ip',
                            returnStdout: true
                        ).trim()
                    }
                    sh 'echo "EC2 Public IP: ${EC2_IP}"'
                }
            }
        }

        stage('Wait for EC2 to be Ready') {
            steps {
                sh '''
                    echo "Waiting 60 seconds for EC2 boot and SSH service..."
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
                    sh '''
                        echo "Generated inventory.ini:"
                        cat inventory.ini
                    '''
                }
            }
        }

        stage('Deploy Nginx using Ansible') {
            steps {
                dir('ansible') {
                    sshagent(credentials: ['ec2-ssh-key']) {
                        sh '''
                            echo "Running Ansible playbook..."
                            export ANSIBLE_HOST_KEY_CHECKING=False
                            ansible-playbook -i inventory.ini playbook.yml
                        '''
                    }
                }
            }
        }

        stage('Show Website URL') {
            steps {
                sh '''
                    echo "========================================"
                    echo "Deployment Successful!"
                    echo "Open this in browser:"
                    echo "http://${EC2_IP}"
                    echo "========================================"
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed. Check Jenkins console logs."
        }
    }
}
