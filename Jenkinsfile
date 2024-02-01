pipeline {
    agent any
    tools {
        terraform 'terraform'
    }

    environment {
        JENKINS_URL = "http://54.156.116.231:8080"
        NODE_NAME = "jnlp-node"
        NODE_DESCRIPTION = "Jenkins Agent Node"
        AWS_REGION = "us-east-1"
        instance_id = ''
        instance_ip = ''
    }

    stages {
        stage('Create Jenkins Agent Node') {
            steps {
                script {
                    sh """
                        curl -O $JENKINS_URL/jnlpJars/jenkins-cli.jar
                        java -jar jenkins-cli.jar -s $JENKINS_URL -webSocket create-node $NODE_NAME <<EOF
                        <slave>
                            <name>$NODE_NAME</name>
                            <description>$NODE_DESCRIPTION</description>
                            <remoteFS>/home/ubuntu</remoteFS>
                            <numExecutors>2</numExecutors>
                            <mode>NORMAL</mode>
                            <retentionStrategy class="hudson.slaves.RetentionStrategy\$Always"/>
                            <launcher class="hudson.slaves.JNLPLauncher" />
                            <label>jnlp</label>
                            <nodeProperties/>
                        </slave>
                        EOF
                    """
                }
            }
        }

        stage('Launch EC2 Instance') {
            steps {
                script {
                    instance_id = sh (
                        script: """
                            aws ec2 run-instances \
                                --image-id ami-0c7217cdde317cfec \
                                --instance-type t2.micro \
                                --key-name nabeel \
                                --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=jnlp-slave}]" \
                                --subnet-id subnet-0badae17f2e23f24a \
                                --security-group-ids sg-03c84415602bb4681 \
                                --region $AWS_REGION \
                                --query 'Instances[0].InstanceId' \
                                --associate-public-ip-address \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()
                    env.instance_id = instance_id

                    sh "aws ec2 wait instance-running --instance-ids $instance_id --region $AWS_REGION"
                    sh "aws ec2 wait instance-status-ok --instance-ids $instance_id --region $AWS_REGION"

                    // public IP of the instance
                    instance_ip = sh (
                        script: """
                            aws ec2 describe-instances \
                                --instance-ids $instance_id \
                                --region $AWS_REGION \
                                --query 'Reservations[0].Instances[0].PublicIpAddress' \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Instance IP: $instance_ip"
                    env.instance_ip = instance_ip

                    sh "curl -O $JENKINS_URL/jnlpJars/agent.jar"

                    withCredentials([sshUserPrivateKey(credentialsId: 'ubuntu', keyFileVariable: 'SSH_KEY', passphraseVariable: '', usernameVariable: 'SSH_USER')]) {
                        sh "scp -o StrictHostKeyChecking=no -i ${SSH_KEY} agent.jar ubuntu@${instance_ip}:/home/ubuntu/agent.jar"
                    }
                }
            }
        }

        stage('Launch Jenkins Agent') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: 'ubuntu', keyFileVariable: 'SSH_KEY', passphraseVariable: '', usernameVariable: 'SSH_USER')]) {
                        sh "ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ubuntu@${instance_ip} 'sudo apt-get update -y'"
                        sh "ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ubuntu@${instance_ip} 'sudo apt install openjdk-11-jre-headless -y'"
                        sh "ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ubuntu@${instance_ip} 'nohup java -jar /home/ubuntu/agent.jar -jnlpUrl $JENKINS_URL/computer/$NODE_NAME/slave-agent.jnlp > /dev/null 2>&1 & disown'"
                    }
                }
            }
        }

        stage('Install Terraform') {
            agent {
                label 'jnlp'
            }
            steps {
                script {
                    withCredentials([sshUserPrivateKey(credentialsId: 'ubuntu', keyFileVariable: 'SSH_KEY', passphraseVariable: '', usernameVariable: 'SSH_USER')]) {
                        sh """
                            terraform --version
                            cd /home/ubuntu/tools/org.jenkinsci.plugins.terraform.TerraformInstallation/terraform
                            sudo mv terraform /usr/local/bin/
                        """
                    }
                }
            }
        }
    }
}
