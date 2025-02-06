pipeline {
    environment{
        IMAGE_NAME="web-app-jenkins"
        TAG="latest"
        STG_URL="ec2-3-86-82-158.compute-1.amazonaws.com"
        PROD_URL="ec2-3-92-132-173.compute-1.amazonaws.com"
        USR_REGISTRY="meskine"
        EXT_PORT="80"
        INT_PORT="5000"
        CONTAINER_NAME="alpine-app"
        DOMAIN="172.17.0.1"
        SSH_USER="ubuntu"
    }

    agent none 
    stages{
        stage("Build"){
            agent any
            steps{
                script{
                    sh '''
                    docker build -t $IMAGE_NAME:$TAG .
                    '''
                }

            }
        }
        stage("Test-Acceptance"){
            agent any 
            steps{
                script{
                    sh '''
                    docker run -d -p $EXT_PORT:$INT_PORT -e PORT=$INT_PORT --name $CONTAINER_NAME $IMAGE_NAME:$TAG
                    sleep 10
                    curl http://$DOMAIN:$EXT_PORT | grep -q 'Hello'
                    sleep 5
                    docker stop $CONTAINER_NAME
                    docker rm   $CONTAINER_NAME
                    '''
                }
            }
        }
        stage('release'){
            agent any 
            environment{
               DOCKERHUB_PWD = credentials('dockerhub-credentials')
            }
            steps{
                script{
                   sh '''
                      echo $DOCKERHUB_PWD | docker login -u $USR_REGISTRY --password-stdin
                      docker tag $IMAGE_NAME:$TAG $USR_REGISTRY/$IMAGE_NAME:$TAG
                      docker push $USR_REGISTRY/$IMAGE_NAME:$TAG
                   '''
                }
            }
        }
        stage("Deploy-staging"){
            agent any 
            when{
              expression { GIT_BRANCH == 'origin/master' }
              
            }
            steps{
                script{
                    echo "deploying to shell-script to ec2"
                    def pullcmd="docker pull $USR_REGISTRY/$IMAGE_NAME:$TAG"
                    def stopcmd=" docker stop $CONTAINER_NAME || echo 'Container not running'"
                    def rmvcmd=" docker rm $CONTAINER_NAME || echo 'Container not found'"
                    def runcmd="docker run -d -p $EXT_PORT:$INT_PORT -e PORT=$INT_PORT --name $CONTAINER_NAME $USR_REGISTRY/$IMAGE_NAME:$TAG"
                    sshagent(['aws-credentials']){
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${STG_URL} ${stopcmd}"
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${STG_URL} ${rmvcmd}"
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${STG_URL} ${pullcmd}"
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${STG_URL} ${runcmd}"
                    }

                }
            }

        }
        stage('test staging'){
            agent any 
            when{
              expression { GIT_BRANCH == 'origin/master' }
             
            }
            steps{
                script{
                    echo " test staging"
                    sh " curl http://${STG_URL}:$EXT_PORT | grep -q 'Hello'"
                }  
            }
        }
        stage("Deploy-prod"){
            agent any
            when{
              expression { GIT_BRANCH == 'origin/master' }
             
            }
            steps{
                script{
                    echo "deploying to shell-script to ec2 production"
                    def stopcmd=" docker stop $CONTAINER_NAME || echo 'Container not running'"
                    def rmvcmd=" docker rm $CONTAINER_NAME || echo 'Container not found'"
                    def pullcmd="docker pull $USR_REGISTRY/$IMAGE_NAME:$TAG"
                    def runcmd="docker run -d -p $EXT_PORT:$INT_PORT -e PORT=$INT_PORT --name $CONTAINER_NAME $USR_REGISTRY/$IMAGE_NAME:$TAG"
                    sshagent(['aws-credentials']){
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${PROD_URL} ${stopcmd}"
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${PROD_URL} ${rmvcmd}"
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${PROD_URL} ${pullcmd}"
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${PROD_URL} ${runcmd}"
                    }

                }
            }

        }
        stage('test production'){
            agent any 
            when{
              expression { GIT_BRANCH == 'origin/master' }
             
            }
            steps{
                script{
                    echo " test production"
                    sh " curl http://${PROD_URL}:$EXT_PORT | grep -q 'Hello'"
                }  
            }
        }
    }
}
