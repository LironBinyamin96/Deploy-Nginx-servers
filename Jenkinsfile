pipeline {
    agent any

    environment {
        VAULT_TOKEN = credentials('vault-token-secret-text')
        VM_HOST = 'Liron.aws.cts.care'
        VM_USER = 'ubuntu'
    }
    
    stages {
        stage('Get SSH Private Key from Vault') {
            steps {
                script {
                    // Create a directory for storing credentials securely
                    sh 'mkdir -p ~/.ssh_temp && chmod 700 ~/.ssh_temp'
                    
                    // Retrieve private key from Vault using direct curl with basic shell processing
                    withCredentials([string(credentialsId: 'vault-token-secret-text', variable: 'VAULT_TOKEN')]) {
                        sh '''
                            # Get response from Vault 
                            RESPONSE=$(curl --silent --header "X-Vault-Token: $VAULT_TOKEN" --request GET http://vault:8200/v1/secret/aws/privat-key)
                
                            # Extract the private key
                            echo "$RESPONSE" | sed 's/.*"value":"//' | sed 's/".*//' > ~/.ssh_temp/ssh_key.pem
                            sed -i 's/\\\\n/\\n/g' ~/.ssh_temp/ssh_key.pem
                            chmod 600 ~/.ssh_temp/ssh_key.pem
                        '''
                    }
                    
                    echo "Private Key retrieved and stored securely"
                    
                    // Test connection to VM
                    sh """
                        ssh -o StrictHostKeyChecking=no -i ~/.ssh_temp/ssh_key.pem ${VM_USER}@${VM_HOST} 'echo "Successfully connected to the virtual machine!"'
                    """
                }
            }
        }
        
        stage('Install Ansible') {
            steps {
                // Install ansible on Jenkins agent if not already installed
                sh '''
                    if ! command -v ansible-playbook &> /dev/null; then
                        sudo apt-get update
                        sudo apt-get install -y ansible
                    fi
                '''
            }
        }

        stage('Install AWS CLI') {
            steps {
                script {
                    // Install AWS CLI
                    sh '''
                        # Install AWS CLI v2
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip awscliv2.zip
                        sudo ./aws/install
                        # Verify installation
                        aws --version
                    '''
                }
            }
        }
        
        stage('Prepare Inventory') {
            steps {
                // Create inventory file for the target VMs dynamically from AWS EC2
                sh '''
                    echo "[webserver]" > inventory
                    INSTANCE_IDS=$(aws ec2 describe-instances --query "Reservations[*].Instances[*].InstanceId" --output text)
                    for INSTANCE_ID in $INSTANCE_IDS; do
                        IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query "Reservations[].Instances[].PublicIpAddress" --output text)
                        echo "$IP ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh_temp/ssh_key.pem ansible_ssh_common_args='-o StrictHostKeyChecking=no'" >> inventory
                    done
                    cat inventory
                '''
            }
        }
        
        stage('Run Ansible Playbook') {
            steps {
                // Run ansible playbook from Jenkins against the remote VM
                sh '''
                    export ANSIBLE_HOST_KEY_CHECKING=False
                    ansible-playbook -i inventory rolling_update.yml -v
                '''
            }
        }
        
        stage('Cleanup') {
            steps {
                // Clean up the SSH key after use
                sh 'rm -rf ~/.ssh_temp'
                echo "Credentials cleaned up"
            }
        }
    }

    post {
        always {
            // Ensure credentials are always cleaned up, even if the pipeline fails
            sh 'rm -rf ~/.ssh_temp || true'
        }
    }
}
