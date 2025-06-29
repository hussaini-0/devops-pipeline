pipeline {
  agent any

  environment {
    TF_IN_AUTOMATION = 'true'
    TF_VAR_admin_username = 'azureuser'
    TF_VAR_admin_ssh_public_key = 'ssh-rsa AAAAB3NzaC1...your_key... azureuser'
    TF_VAR_location = 'East US'
    TF_VAR_resource_group_name = 'devops-pipeline-rg'
    TF_VAR_vm_name = 'devops-vm'
  }

  stages {
    stage('Terraform Init') {
      steps {
        withCredentials([
          string(credentialsId: 'ARM_CLIENT_ID', variable: 'ARM_CLIENT_ID'),
          string(credentialsId: 'ARM_CLIENT_SECRET', variable: 'ARM_CLIENT_SECRET'),
          string(credentialsId: 'ARM_TENANT_ID', variable: 'ARM_TENANT_ID'),
          string(credentialsId: 'ARM_SUBSCRIPTION_ID', variable: 'ARM_SUBSCRIPTION_ID')
        ]) {
          withEnv([
            "TF_VAR_client_id=${ARM_CLIENT_ID}",
            "TF_VAR_client_secret=${ARM_CLIENT_SECRET}",
            "TF_VAR_tenant_id=${ARM_TENANT_ID}",
            "TF_VAR_subscription_id=${ARM_SUBSCRIPTION_ID}"
          ]) {
            dir('terraform') {
              sh 'terraform init -input=false'
            }
          }
        }
      }
    }

    stage('Terraform Import') {
      steps {
        withCredentials([
          string(credentialsId: 'ARM_SUBSCRIPTION_ID', variable: 'ARM_SUBSCRIPTION_ID')
        ]) {
          dir('terraform') {
            sh '''
              terraform import azurerm_linux_virtual_machine.main "/subscriptions/$ARM_SUBSCRIPTION_ID/resourceGroups/devops-pipeline-rg/providers/Microsoft.Compute/virtualMachines/devops-vm" || true
              terraform import azurerm_network_interface_security_group_association.main "/subscriptions/$ARM_SUBSCRIPTION_ID/resourceGroups/devops-pipeline-rg/providers/Microsoft.Network/networkInterfaces/nic-devops/networkSecurityGroups/nsg-devops" || true
            '''
          }
        }
      }
    }

    stage('Terraform Apply') {
      steps {
        withCredentials([
          string(credentialsId: 'ARM_CLIENT_ID', variable: 'ARM_CLIENT_ID'),
          string(credentialsId: 'ARM_CLIENT_SECRET', variable: 'ARM_CLIENT_SECRET'),
          string(credentialsId: 'ARM_TENANT_ID', variable: 'ARM_TENANT_ID'),
          string(credentialsId: 'ARM_SUBSCRIPTION_ID', variable: 'ARM_SUBSCRIPTION_ID')
        ]) {
          withEnv([
            "TF_VAR_client_id=${ARM_CLIENT_ID}",
            "TF_VAR_client_secret=${ARM_CLIENT_SECRET}",
            "TF_VAR_tenant_id=${ARM_TENANT_ID}",
            "TF_VAR_subscription_id=${ARM_SUBSCRIPTION_ID}",
            "TF_VAR_admin_username=azureuser",
            "TF_VAR_admin_ssh_public_key=ssh-rsa AAAAB3NzaC1...your_key... azureuser",
            "TF_VAR_location=East US",
            "TF_VAR_resource_group_name=devops-pipeline-rg",
            "TF_VAR_vm_name=devops-vm"
          ]) {
            dir('terraform') {
              sh 'terraform apply -input=false -auto-approve'
            }
          }
        }
      }
    }

    stage('Get Public IP from Terraform Output') {
      steps {
        script {
          env.VM_PUBLIC_IP = sh(
            script: 'terraform -chdir=terraform output -raw vm_ip',
            returnStdout: true
          ).trim()
          echo "Azure VM IP: ${env.VM_PUBLIC_IP}"
        }
      }
    }

    stage('Run Ansible Playbook') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ansible_ssh_key', keyFileVariable: 'SSH_KEY')]) {
          sh '''
            export ANSIBLE_HOST_KEY_CHECKING=False
            echo "[web]" > inventory.ini
            echo "$VM_PUBLIC_IP ansible_user=azureuser ansible_ssh_private_key_file=$SSH_KEY" >> inventory.ini

            ansible-playbook ansible/install_web.yml -i inventory.ini
            rm -f inventory.ini
          '''
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        sh 'curl -s http://$VM_PUBLIC_IP | grep -i "devops project completed"'
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline executed successfully!"
    }
    failure {
      echo "❌ Pipeline failed. Check logs and validate stages."
    }
  }
}
