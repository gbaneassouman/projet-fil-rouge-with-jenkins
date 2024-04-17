/* groovylint-disable CompileStatic, NglParseError */
/* groovylint-disable DuplicateListLiteral, DuplicateStringLiteral, GStringExpressionWithinString, LineLength, NestedBlockDepth, NglParseError */
/* groovylint-disable-next-line CompileStatic */
/* groovylint-disable-next-line CompileStatic, NglParseError */
/* import shared library */
/* groovylint-disable-next-line CompileStatic */
/* groovylint-disable-next-line CompileStatic */

@Library('slack-shared-library') _

pipeline {
    environment {
        DOCKERHUB_PASSWORD  = credentials('DOCKER_HUB_ID')
        IMAGE_TAG = '1.0'
        IMAGE_NAME = 'ic-webapp'
        HOST_PORT = '8000'
        CONTAINER_PORT = '8080'
        TEST_CONTAINER = 'test-ic-webapp'
        DOCKER_HUB = 'openlab89'
        TF_DIR = '/var/lib/jenkins/workspace/projet-fil-rouge/src/terraform'
        ANS_DIR = '$WORKSPACE/src/ansible'
        IP_FILE = 'src/terraform/$ENV_NAME/files/infos_ec2.txt'
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_PRIVATE_KEY = credentials('AWS_SSH_KEY')
        AWS_KEY_NAME = 'devops-gbane'
    }
    agent any //declaration globale de l'agent
    stages {
        stage('Cloning code') {
            steps {
                script {
                    sh '''
                         rm -rf projet-fil-rouge-with-jenkins || echo "Directory doesn't exists "
                         sleep 2
                         git clone https://github.com/gbaneassouman/projet-fil-rouge-with-jenkins.git
                     '''
                }
            }
        }
        stage('Build image') {
            steps {
                script {
                     /* groovylint-disable-next-line GStringExpressionWithinString */
                    sh '''
                       docker build --no-cache -f ./src/Dockerfile -t ${IMAGE_NAME}:${IMAGE_TAG} ./src/
                    '''
                }
            }
        }
        stage('Test image') {
            steps {
                script {
                    /* groovylint-disable-next-line GStringExpressionWithinString */
                    sh 'docker stop ${TEST_CONTAINER} || true && docker rm ${TEST_CONTAINER} || true'
                    sh 'docker run --name ${TEST_CONTAINER} -d -p ${HOST_PORT}:${CONTAINER_PORT} -e PORT=${CONTAINER_PORT} ${IMAGE_NAME}:${IMAGE_TAG}'
                    sh 'sleep 10'
                    sh 'curl -k http://172.17.0.1:${HOST_PORT}| grep -i "Odoo"'
                }
            }
        }
        stage('Release image') {
            steps {
                script {
                    /* groovylint-disable-next-line GStringExpressionWithinString */
                    sh '''
                        docker stop ${TEST_CONTAINER} || true && docker rm ${TEST_CONTAINER} || true
                        docker save ${IMAGE_NAME}:${IMAGE_TAG} > /tmp/${IMAGE_NAME}:${IMAGE_TAG}.tar
                        docker image tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_HUB}/${IMAGE_NAME}:${IMAGE_TAG}
                        echo $DOCKERHUB_PASSWORD_PSW | docker login -u ${DOCKER_HUB} --password-stdin
                        docker push ${DOCKER_HUB}/${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }
        // Ce stage permet de generer le fichier .env necéssaire à docker-compose.yml
        stage('Generate env-file') {
            environment {
                ODOO_USER = credentials('ODOO_USER')
                ODOO_DB = credentials('ODOO_DB')
                ODOO_PASSWORD = credentials('ODOO_PASSWORD')
                POSTGRES_DB = credentials('POSTGRES_DB')
                POSTGRES_PASSWORD = credentials('POSTGRES_PASSWORD')
                POSTGRES_USER = credentials('POSTGRES_USER')
                PGDATA = credentials('PGDATA')
                PGADMIN_DEFAULT_EMAIL = credentials('PGADMIN_DEFAULT_EMAIL')
                PGADMIN_DEFAULT_PASSWORD = credentials('PGADMIN_DEFAULT_PASSWORD')
                PGADMIN_LISTEN_PORT = credentials('PGADMIN_LISTEN_PORT')
            }
            steps{
                script {
                    sh '''
                        echo "Generating env-file"
                        touch ./src/.env
                        echo USER=$ODOO_USER >> ./src/.env
                        echo ODOO_DB=$ODOO_DB >> ./src/.env
                        echo PASSWORD=$ODOO_PASSWORD >> ./src/.env
                        echo POSTGRES_DB=$POSTGRES_DB >> ./src/.env
                        echo POSTGRES_PASSWORD=$POSTGRES_PASSWORD >> ./src/.env
                        echo POSTGRES_USER=$POSTGRES_USER >> ./src/.env
                        echo PGDATA=$PGDATA >> ./src/.env
                        echo PGADMIN_DEFAULT_EMAIL=$PGADMIN_DEFAULT_EMAIL >> ./src/.env
                        echo PGADMIN_DEFAULT_PASSWORD=$PGADMIN_DEFAULT_PASSWORD >> ./src/.env
                        echo PGADMIN_LISTEN_PORT=$PGADMIN_LISTEN_PORT >> ./src/.env
                        echo $WORKSPACE
                    '''
                }
            }
        }
        stage('Staging ec2') {
            environment {
                ENV_NAME = 'staging'
                AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
                AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
                AWS_PRIVATE_KEY = credentials('AWS_SSH_KEY')
                AWS_KEY_NAME = 'devops-gbane'
                AWS_PROFILE = 'default'
            }
            steps {
                script {
                    /* groovylint-disable-next-line GStringExpressionWithinString */
                    sh '''
                        echo "Generating aws credentials"
                        rm -rf devops-gbane.pem ~/.aws
                        mkdir -p ~/.aws
                        echo "[default]" > ~/.aws/credentials
                        echo "aws_access_key_id=$AWS_ACCESS_KEY_ID" >> ~/.aws/credentials
                        echo "aws_secret_access_key=$AWS_SECRET_ACCESS_KEY" >> ~/.aws/credentials
                        echo "aws_profile=$AWS_PROFILE" >> ~/.aws/credentials
                        rm -rf $TF_DIR/staging/files/$AWS_KEY_NAME.pem
                        cat "$AWS_PRIVATE_KEY" > $TF_DIR/staging/files/$AWS_KEY_NAME.pem
                        chmod 400 $TF_DIR/staging/files/$AWS_KEY_NAME.pem
                        chmod 400 ~/.aws/credentials
                        cd src/terraform/$ENV_NAME
                        terraform init -input=false
                        terraform plan -out $ENV_NAME.plan
                        terraform apply $ENV_NAME.plan
                    '''
                }
            }
        }
        stage('Deploy apps to staging') {
            environment {
                ENV_NAME = 'staging'
            }
            steps {
                script {
                    //si on l'execute dans un script le stage n'echoura jamais si le deploy echoue
                    sh '''
                        #!/bin/bash
                        INFILE=$PWD/list.txt
                        # Read the input file line by line
                        mkdir -p app-dir
                        export instance_ip=$(awk '{print $1}' src/terraform/staging/files/infos_ec2.txt)
                        for LINE in $(cat "$INFILE")
                        do
                            cp -r src/"$LINE" app-dir/
                        done
                        cp src/scripts/deploy-apps.sh app-dir/ && cp src/terraform/staging/files/infos_ec2.txt app-dir/
                        zip -r app-dir.zip app-dir/
                        scp -i $TF_DIR/$ENV_NAME/files/$AWS_KEY_NAME.pem -o StrictHostKeyChecking=no -r app-dir.zip ubuntu@$instance_ip:~/
                        ssh -i $TF_DIR/$ENV_NAME/files/$AWS_KEY_NAME.pem -o StrictHostKeyChecking=no  ubuntu@$instance_ip "unzip ~/app-dir.zip"
                        ssh -i $TF_DIR/$ENV_NAME/files/$AWS_KEY_NAME.pem -o StrictHostKeyChecking=no  ubuntu@$instance_ip "chmod +x ~/app-dir/deploy-apps.sh"
                        ssh -i $TF_DIR/$ENV_NAME/files/$AWS_KEY_NAME.pem -o StrictHostKeyChecking=no  ubuntu@$instance_ip "cd ~/app-dir && sh deploy-apps.sh"
                        rm -rf app-dir app-dir.zip
                    '''
                }
            }
        }
    }
    post {
        always {
            script {
                /* Use Slack-notification.groovy from shared library */
                slackNotifier currentBuild.result
            }
        }
    }
}