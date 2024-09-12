# Jenkins Active Reactive Choice Parameter Configuration and Script Deployment on Server 

### In this task, I have created two parameters in Jenkins pipeline.
### One parameter, lists all the folders in the S3 bucket and other parameter takes the reference from the first parameter and lists all the files within that selected folder.

### Not only this, but the selected script will be deployed on an EC2 instance and will be executed over there as well.

### Prerequisites:
```
Plugins:

1. Active Choices Plug-in
2. Groovy Postbuild
3. Post build task
4. PostBuildScript Plugin
5. Build Name and Description Setter
6. AnsiColor


> Jenkins user on the server must have aws credentials configured:
su - jenkins
aws configure

SSH must be configured with the EC2 instance using Jenkins user.
```

### So, first of all, create a pipeline > go to configuration > Select "This project is parameterised"

```
Name: FOLDER_LIST
```
### Select Groovy Script

```
def command = ['bash', '-c', '''
    aws s3api list-objects --bucket <bucket_name> --region <region_name> --query 'Contents[].{Key: Key}' --output text | grep '/' | awk -F'/' '{print $1}' | sort | uniq ''']
def list_command = command.execute()
def FOLDER_LIST = list_command.text.tokenize()
return FOLDER_LIST
```
```
Choice Type: Single Select
Filter starts at: 1
```

### > Add Parameter: Active Choices Reactive Parameter
### Select Groovy Script

```
def greeting = "${FOLDER_LIST}"

def command2 = ['bash', '-c', """
       aws s3api list-objects --bucket <bucket_name> --region <region_name> --query "Contents[?starts_with(Key, '${greeting}')].Key" --output text | awk -v prefix='${greeting}' '{sub(prefix, "", \$0); print}' | tr '\t' '\n' | sed 's|.*/||' | sed 's/.sh\$//'
     """]

def process = command2.execute()
def OBJECT_LIST = process.text.tokenize('\n')

return OBJECT_LIST
```

```
Choice Type: Single Select

Reference parameters: FOLDER_LIST

Filter Starts At: 1
```

### This set-up must show the list of all the folders and files accordingly.


### Now, it's time to write a pipeline script.
### Before that, you need to add the server ip address as username and its pem file in Jenkins Credentials.

```
Dashboard > Manage Jenkins > Credentials > System > Global Credentials > SSH Username with private key

Scope: Global
ID: ec2-ssh
username: <server_IP>
Private Key: Copy and paste the pem file
```

### Below is the pipeline script:
```
pipeline {
    agent any
    environment {
        S3_BUCKET = '<bucket_name>'
        EC2_INSTANCE_IP = '<server_IP>'
        SSH_CREDENTIALS_ID = 'ec2-ssh'
    }
    stages {
        stage('Debug Parameters') {
            steps {
                ansiColor('xterm') {
                    script {
                        def red = '\033[1;31m'
                        def reset = '\033[0m'

                        echo "The name of selected folder is ${red}${params.FOLDER_NAME}${reset} and the name of selected file is ${red}${params.FILE_NAME}${reset}"

                        timeout(time: 20, unit: 'SECONDS') {
                            def userInput = input message: 'Do you want to proceed with the script?', 
                                                parameters: [choice(name: 'Proceed?', choices: ['YES', 'NO'], description: 'Select YES to continue or NO to abort')]

                            if (userInput == 'NO') {
                                error 'Pipeline aborted by user.'
                            }
                        }
                    }
                }
            }
        }
        stage('Copy Script') {
            when {
                expression {
                    return currentBuild.result == null || currentBuild.result == 'SUCCESS'
                }
            }
            steps {
                script {
                    def folder = params.FOLDER_NAME
                    def file = params.FILE_NAME
                    sh 'aws --version'

                    sh "aws s3 cp s3://${S3_BUCKET}/${folder}/${file}.sh ./${file}.sh"

                    sh "ls -l ./${file}.sh"
                    sh "file ./${file}.sh"

                    withCredentials([sshUserPrivateKey(credentialsId: SSH_CREDENTIALS_ID, keyFileVariable: 'SSH_KEY')]) {
                        sh "ssh-keyscan -H ${EC2_INSTANCE_IP} >> ~/.ssh/known_hosts"
                        sh "scp -i ${SSH_KEY} ./${file}.sh ubuntu@${EC2_INSTANCE_IP}:/home/ubuntu/"
                        sh """
                        ssh -i ${SSH_KEY} ubuntu@${EC2_INSTANCE_IP} << 'EOF'
                        chmod u+x /home/ubuntu/${file}.sh
                        echo -e "/var/lib/jenkins/orin\ny" | /home/ubuntu/${file}.sh
                        EOF
                        """
                    }
                }
            }
        }

        stage('Echo Script Execution') {
            steps {
                ansiColor('xterm') {
                    script {
                        def red = '\033[1;31m'
                        def reset = '\033[0m'
                        echo "The script ${red}${params.FILE_NAME}${reset} has been successfully copied and executed."
                    }
                }
            }
        }
    }
}
```

### End!
