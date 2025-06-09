def DOCKER_SERVER="10.10.4.130"  //Provide Docker Server Public IP/Private IP Address.
pipeline {
    agent {
        label{
            label "Slave-1"
            customWorkspace "/home/jenkins/bankapp"
        }
    }
    environment{
        JAVA_HOME="/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        PATH="$PATH:$JAVA_HOME/bin:/opt/apache-maven/bin:/opt/node-v16.0.0/bin:/usr/local/bin"
        MYSQL_ROOT_PASSWORD=credentials('mysql_root_password')
        MYSQL_DATABASE=credentials('mysql_database')
        newrelic_license_key=credentials('newrelic_license_key')
    }
    parameters { 
        string(name: 'COMMIT_ID', defaultValue: '', description: 'Provide the Commit ID') 
        string(name: 'REPO_NAME', defaultValue: '', description: 'Provide the ECR Repository Name for Application Image')
        string(name: 'TAG_NAME', defaultValue: '', description: 'Provide the TAG Name')
    }
    stages{
        stage("clone-code"){
            steps{
                cleanWs()
                checkout scmGit(branches: [[name: '${COMMIT_ID}']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-cred', url: 'https://github.com/singhritesh85/Bank-App-Monitoring-Logging-using-NewRelic-Private.git']])
            }
        }
        stage("Build and DockerImage"){
            steps{
                sh 'mvn clean install'
                sh 'wget https://download.newrelic.com/newrelic/java-agent/newrelic-agent/current/newrelic-java.zip'
                sh 'unzip newrelic-java.zip'
                sh "yes|cp ./newrelic.yml ./newrelic/"
                sh "sed -i -e ''s/##MYSQL_ROOT_PASSWORD##/${MYSQL_ROOT_PASSWORD}/g'' ./.env"
                sh "sed -i -e ''s/##MYSQL_DATABASE##/${MYSQL_DATABASE}/g'' ./.env"
                sh "sed -i -e ''s@##REPO_NAME##@${REPO_NAME}@g'' ./.env"
                sh "sed -i -e ''s/##TAG_NAME##/${TAG_NAME}/g'' ./.env"
                sh "sed -i -e ''s/'<%= license_key %>'/$newrelic_license_key/g'' ./newrelic/newrelic.yml"
                sh "sed -i -e ''s/'app_name: My Application'/'app_name: bankapp'/g'' ./newrelic/newrelic.yml"
                sh "echo -e ''enable_process_metrics: true'' >> ./newrelic/newrelic.yml"
                sh "echo -e ''status_server_enabled: true'' >> ./newrelic/newrelic.yml"
                sh "echo -e ''status_server_port: 18003'' >> ./newrelic/newrelic.yml"
                sh "echo -e ''license_key: $newrelic_license_key'' >> ./newrelic/newrelic.yml"
                sh "echo -e ''custom_attributes:'' >> ./newrelic/newrelic.yml"
                sh "echo -e '  nr_deployed_by: bankapp' >> ./newrelic/newrelic.yml"
                sh "scp -rv Dockerfile-Project-1 docker-admin@${DOCKER_SERVER}:~/"
                sh "scp -rv target docker-admin@${DOCKER_SERVER}:~/"
                sh "scp -rv docker-compose.yaml docker-admin@${DOCKER_SERVER}:~/"
                sh "scp -rv .env docker-admin@${DOCKER_SERVER}:~/"
                sh "scp -rv newrelic docker-admin@${DOCKER_SERVER}:~/"
                sh 'docker system prune -f'
                sh "docker build -t ${REPO_NAME}:${TAG_NAME} -f Dockerfile-Project-1 ."
                sh 'aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 027330342406.dkr.ecr.us-east-2.amazonaws.com'
                sh "docker push ${REPO_NAME}:${TAG_NAME}"
            }
        }
        stage("Deployment"){
            steps{
                sh """ssh docker-admin@$DOCKER_SERVER <<-EOF
                docker-compose -p bankapp down     ### do not use docker-compose down -v here oherwise docker volume will be deleted. 
                docker system prune -f
                sudo docker-compose -p bankapp up -d 
                exit
                EOF
                """
            }
        }
    }
}
