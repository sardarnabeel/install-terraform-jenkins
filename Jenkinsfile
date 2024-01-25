pipeline {
    agent any
    
    environment {
        JENKINS_URL = "http://52.91.1.20:8080"
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
                            <label></label>
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
                                --subnet-id subnet-09458ae752232b41f \
                                --security-group-ids sg-09eb5b422b44b8a07 \
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

                    //public IP of the instance
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

                    sh "scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/workspace/jenkinsfile/nabeel.pem agent.jar ubuntu@$instance_ip:/home/ubuntu/agent.jar"
                }
            }
        }

        stage('Launch Jenkins Agent') {
            steps {
                script {

                    sh """
                        ssh -v -o StrictHostKeyChecking=no -i /var/lib/jenkins/workspace/jenkinsfile/nabeel.pem ubuntu@$instance_ip \
                            "sudo apt-get update &&
                            sudo apt-get install awscli -y &&
                            sudo apt-get install -y gnupg software-properties-common &&
                            echo 'gnupg package isntalled' &&
                            wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg &&
                            echo 'terraform download' &&
                            echo 'deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com jammy main' | sudo tee /etc/apt/sources.list.d/hashicorp.list &&
                            echo 'command executed successfully' &&
                            sudo apt-get update -y &&
                            echo 'update successfully' &&
                            sudo apt-get install -y terraform &&
                            echo 'terraform installed' &&
                            which terraform &&
                            sudo apt install openjdk-11-jre-headless -y &&
                            echo 'java installed' &&
                            java -jar /home/ubuntu/agent.jar -jnlpUrl $JENKINS_URL/computer/$NODE_NAME/slave-agent.jnlp > /dev/null 2>&1 &"
                    """
                    
                }
            }
            post {
               always {
                  script {
                      echo "Executing post-action"
                      echo "Instance ID: $instance_id"
                      echo "Instance IP: $instance_ip"

                      sh "aws ec2 terminate-instances --instance-ids $instance_id --region $AWS_REGION"
            
                      sh "java -jar jenkins-cli.jar -s $JENKINS_URL -webSocket delete-node $NODE_NAME"
                  }
               } 
            }
        }
        // stage('Install Terraform') {
        //     steps {
        //         script {
        //             sh """
        //                 ssh -v -o StrictHostKeyChecking=no -i /var/lib/jenkins/workspace/jenkinsfile/nabeel.pem ubuntu@${instance_ip} '
        //                     sudo apt-get install -y gnupg software-properties-common &&
        //                     echo "gnupg package installed" &&
        //                     wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg &&
        //                     echo "terraform download" &&
        //                     echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com jammy main" | sudo tee /etc/apt/sources.list.d/hashicorp.list &&
        //                     echo "command executed successfully" &&
        //                     sudo apt-get update -y &&
        //                     echo "update successfully" &&
        //                     sudo apt-get install -y terraform &&
        //                     echo "terraform installed" &&
        //                     which terraform || echo "Terraform installation failed. Check logs for more details."
        //                 '
        //             """
        //         }
        //     }
        // }
    }
    // post {
    //     always {
    //         script {

    //           sh "aws ec2 terminate-instances --instance-ids $instance_id --region $AWS_REGION"
            
    //           sh "java -jar jenkins-cli.jar -s $JENKINS_URL -webSocket delete-node $NODE_NAME"
    //         }
    //     }
    // }

    // post {
    //     always {
    //        script {
    //           echo "Executing post-action"
    //           echo "Instance ID: $instance_id"
    //           echo "Instance IP: $instance_ip"

    //           sh "aws ec2 terminate-instances --instance-ids $instance_id --region $AWS_REGION"
            
    //           sh "java -jar jenkins-cli.jar -s $JENKINS_URL -webSocket delete-node $NODE_NAME"
    //        }
    //     } 
    //  }
}
