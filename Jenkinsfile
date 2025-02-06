pipeline {
    environnement{
        IMAGE_NAME="web-app-jenkins"
        IMAGE_TAG="latest"
        STG_URL=$URL_PREPROD
        PROD_URL=$URL_PROD
        USR_REGISTRY="meskine"
        EXT_PORT="80"
        INT_PORT=$PORT
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
                    docker save -o alpineweb.war $IMAGE_NAME:$TAG
                    '''
                }

            }
        }
        stage("Test-Acceptance"){
            agent none 
            steps{
                script{
                    sh '"
                    
                    docker run -d -p $EXT_PORT:$INT_PORT -e PORT=$INT_PORT --name $CONTAINER_NAME $IMAGE_NAME:$TAG
                    sleep 10
                    curl http://$DOMAIN:$EXT_PORT | grep -q 'Hello'
                    sleep 5
                    docker stop $CONTAINER_NAME
                    docker rm   $CONTAINER_NAME
                    "'
                }
            }
        }
        stage('release'){
            agent none 
            environnement{
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
            agent none 
            environnement{
              expression { GIT_BRANCH == 'origin/master' }
            }
            steps{
                script{
                    echo "deploying to shell-script to ec2"
                    def pullcmd="docker push $USR_REGISTRY/$IMAGE_NAME:$TAG"
                    def runcmd="docker run -d -p $EXT_PORT:$INT_PORT -e PORT=$INT_PORT --name $CONTAINER_NAME $IMAGE_NAME:$TAG"
                    sshagent(['ec2']){
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${STG_URL} ${pullcmd}"
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${STG_URL} ${runcmd}"
                    }

                }
            }

        }
        stage("Deploy-prod"){
            agent none 
            environnement{
              expression { GIT_BRANCH == 'origin/master' }
            }
            steps{
                script{
                    echo "deploying to shell-script to ec2 production"
                    def pullcmd="docker push $USR_REGISTRY/$IMAGE_NAME:$TAG"
                    def runcmd="docker run -d -p $EXT_PORT:$INT_PORT -e PORT=$INT_PORT --name $CONTAINER_NAME $IMAGE_NAME:$TAG"
                    sshagent(['ec2']){
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${PROD_URL} ${pullcmd}"
                       sh "ssh -o StrictHostKeyChecking=no $SSH_USER@${PROD_URL} ${runcmd}"
                    }

                }
            }

        }
    }
}
