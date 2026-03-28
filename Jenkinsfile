pipeline {
    agent any

    // Parameter to choose action: deploy or destroy
    parameters {
        choice(name: 'ACTION', choices: ['deploy', 'destroy'], description: 'Choose action: deploy or destroy')
    }

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

        stage('Terraform Deploy/Destroy') {
            steps {
                dir('terraform') {
                    script {
                        if (params.ACTION == 'deploy') {
                            sh '''
                                echo "Deploying resources with Terraform..."
                                terraform apply -auto-approve -var="key_name=server"
                            '''
                            env.EC2_IP = sh(
                                script: 'terraform output -raw public_ip',
                                returnStdout: true
                            ).trim()
                            sh 'echo "EC2 Public IP: ${EC2_IP}"'
                        } else if (params.ACTION == 'destroy') {
                            sh '''
                                echo "Destroying Terraform resources..."
                                terraform destroy -auto-approve -var="key_name=server"
                            '''
                        }
                    }
                }
            }
        }

        stage('Wait for EC2 to be Ready') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                sh '''
                    echo "Waiting 60 seconds for EC2 boot and SSH service..."
                    sleep 60
                '''
            }
        }

        stage('Create Ansible Inventory') {
            when {
                expression { params.ACTION == 'deploy' }
            }
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
            when {
                expression { params.ACTION == 'deploy' }
            }
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
            when {
                expression { params.ACTION == 'deploy' }
            }
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
