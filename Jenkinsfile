def getEnvVar(String deployEnv, String paramName){
    return sh (script: "grep '${paramName}' env_vars/${deployEnv}.properties|cut -d'=' -f2", returnStdout: true).trim();
}

pipeline{

agent any

options {
      timeout(time: 1, unit: 'HOURS') 
}

parameters {
    password(name:'AWS_KEY', defaultValue: '', description:'Enter AWS_KEY')
    choice(name: 'DEPLOY_ENV', choices: ['dev','sit','uat','prod'], description: 'Select the deploy environment')
    choice(name: 'ACTION_TYPE', choices: ['deploy','create','destroy'], description: 'Create or destroy')
    string(name: 'INSTANCE_TYPE', defaultValue: 't2.large', description: 'Type of instance')
    string(name: 'STACK_NAME', defaultValue: 'atk-test', description: 'Unique name of stack')
    string(name: 'SPOT_PRICE', defaultValue: '0.037', description: 'Spot price')
    string(name: 'AWS_DEFAULT_REGION', defaultValue: 'ap-southeast-1', description: 'AWS default region')
    string(name: 'PLAYBOOK_TAGS', defaultValue: 'all', description: 'playbook tags to run')
}

stages{
    stage('Init'){
        steps{
            checkout scm 
        script{
        env.DEPLOY_ENV = "$params.DEPLOY_ENV"
        env.ACTION_TYPE = "$params.ACTION_TYPE"
        env.INSTANCE_TYPE = "$params.INSTANCE_TYPE"
        env.SPOT_PRICE = "$params.SPOT_PRICE"
        env.PLAYBOOK_TAGS = "$params.PLAYBOOK_TAGS"
        env.STACK_NAME = "$params.STACK_NAME"
        env.AWS_DEFAULT_REGION = "$params.AWS_DEFAULT_REGION"
        env.APP_ID = getEnvVar("${env.DEPLOY_ENV}",'APP_ID')
        env.repo_bucket_credentials_id = "ec2s3admin";
        env.AMI_ID = "ami-8e0205f2";
        env.aws_s3_bucket_name = 'jvcdp-repo';
        env.aws_s3_bucket_region = 'ap-southeast-1';
        env.APP_BASE_DIR = pwd()
        env.GIT_HASH = sh (script: "git rev-parse --short HEAD", returnStdout: true)
        env.TIMESTAMP = sh (script: "date +'%Y%m%d%H%M%S%N' | sed 's/[0-9][0-9][0-9][0-9][0-9][0-9]\$//g'", returnStdout: true)
        }
        echo "do some init here";

        }
    }

    stage('Create Stack'){
        when {
        expression {
            return env.ACTION_TYPE == 'create';
            }
        }
        steps{
            script{
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
                accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
                credentialsId: "${repo_bucket_credentials_id}", 
                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]){
                    sh '''#!/bin/bash -xe
cat <<EOF > cf-params.json
[
    { "ParameterKey": "KeyName",
    "ParameterValue": "cdhstack_admin"
    },
    {"ParameterKey": "InstanceType",
    "ParameterValue": "${INSTANCE_TYPE}"
    },
    {"ParameterKey": "ImageId",
        "ParameterValue": "${AMI_ID}"
    }
]
EOF
                    aws  cloudformation create-stack --stack-name=${STACK_NAME} --template-body file://cloudformation-stack.yml --parameters file://cf-params.json
                    aws  cloudformation wait stack-create-complete --stack-name=${STACK_NAME}
                    aws  cloudformation describe-stacks  --stack-name=${STACK_NAME}

cat << EOF > hosts
[all:vars]
ansible_ssh_user: centos
ansible_ssh_private_key_file: ~/.ssh/cdhstack_admin.pem
ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
ansible_become_user: root
ansible_become_method: sudo

[ec2]
EOF

                    aws  ec2 describe-instances --filters Name=tag:Name,Values=${STACK_NAME} --query "Reservations[*].Instances[*].PublicIpAddress" --output=text >> hosts
                    cat hosts
                    '''
                }
            }
        }
    }
    stage('Deploy'){
        when {
        expression {
            return env.ACTION_TYPE == 'deploy';
            }
        }
        steps{
        sh '''
        cd $APP_BASE_DIR
        ansible-playbook -vvvv -i hosts --tags $PLAYBOOK_TAGS main.yml
        '''
        }
    }
    stage('Destroy Stack'){
        when {
        expression {
            return env.ACTION_TYPE == 'destroy';
            }
        }
        steps{
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
                accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
                credentialsId: "${repo_bucket_credentials_id}", 
                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]){
            sh '''
            aws  cloudformation delete-stack --stack-name atk-test
            aws  cloudformation wait stack-delete-complete --stack-name=${STACK_NAME}
            '''
            }
        }
    }
}

post {
    always {
        sh '''
        echo "perform some cleanup"
        rm -f $APP_BASE_DIR/cf-params.json | true
        '''
    }
}
}
