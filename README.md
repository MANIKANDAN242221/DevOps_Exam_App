Explaination In Pipeline Script
----------------------------------------------------------------
#Agent
pipeline{
      agent any
      }
#the comment using run on any available jenkins node/agent 

#Enviroment 
environment {
    DOCKER_IMAGE = "kastrov/devopsexamapp:latest"
}
# Environment is Give the variable of docker images 


#Checkout
stage('Git Checkout') {
    steps {
        git url: 'https://github.com/KastroVKiran/devops-exam-app.git', 
            branch: 'master'
    }
}

# In checkout using for clone the code in GitHub repo on main to Jenkins Woker node path


#Docker Verify 
stage('Verify Docker Compose') {
    steps {
        sh '''
        docker compose version || { echo "Docker Compose not available"; exit 1; }
        '''
    }
}

# This stage is if Check docker compose is installed 
# In this stage scrpit will be Fail you not installed Docker Compose 


#Build Docker Image 

stage('Build Docker Image') {
    steps {
        dir('backend') {
            script {
                withDockerRegistry(credentialsId: 'docker-creds', toolName: 'docker') {
                    sh "docker build -t ${DOCKER_IMAGE} ."
                }
            }
        }
    }
}

# In This script we frist Move into the Backend Directory ,
#Uses withDockerRegistry to login to Docker Hub using docker-creds 
# And Then Build Docker image with tag latest Image 


#Docker Push
stage('Push to Docker Hub') {
    steps {
        script {
            withDockerRegistry(credentialsId: 'docker-creds', toolName: 'docker') {
                sh """
                docker tag ${DOCKER_IMAGE} ${DOCKER_IMAGE}
                docker push ${DOCKER_IMAGE}
                """
            }
        }
    }
}

#Again login Docker Hub , Push the latest image in Docker Hub 


#Deploy With Docker Compose 

stage('Deploy With Docker Compose') {
    steps {
        sh ''' 
        docker compose down --remove-orphans || true
        docker compose up -d
         echo "Waiting for MySQL to be ready..."
        timeout 120s bash -c '
        while ! docker compose exec -T mysql mysqladmin ping -uroot -prootpass --silent;
        do 
            sleep 5;
            docker compose logs mysql --tail=5 || true;
        done'
        sleep 10
        '''
        }
      }

#Stop existing Container, Start Dcoker compose service in Background -d
# Is there Start in Mysql Process
#Waits up to 120 seconds for MySQL to respond to mysqladmin ping.
#Sleeps extra 10 seconds to ensure DB is fully ready.



#Verify Deployment the container

stage('Verify Deployment') {
    steps {
        sh '''
        echo "=== Container Status ==="
        docker compose ps -a
        echo "=== Testing Flask Endpoint ==="
        curl -I http://localhost:5000 || true
        '''
    }
}

# Print The Conatiner status ( docker compose ps -a )
# Make the HTTP request to local curl -I http://localhost:5000 Check to Flash App respones


#Post Block 

success {
    echo '🚀 Deployment successful!'
    sh 'docker compose ps'
    sh 'docker images | grep devopsexamapp'
}

#Print the Success Message , When the pipeline success fully compelety the msg will e print 


failure {
    echo '❗ Pipeline failed. Check logs above.'
    sh '''
    echo "=== Error Investigation ==="
    docker compose logs --tail=50 || true
    '''
}

#Pirnt the error message 

always {
    sh '''
    echo "=== Final Logs ==="
    docker compose logs --tail=20 || true
    '''
}

# Always prints the last 20 lines of all logs for a final overview.









