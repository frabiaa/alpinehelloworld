
/* import library */ 
// @Library('slack_Notifier')_

pipeline{
    environment{
        IMAGE_NAME = "alpinehelloworld"
        IMAGE_TAG = "${BUILD_TAG}"
        USERNAME = "frabiaa"
        CONTAINER_NAME = "alpinehelloworld"
        STAGING = "rabiaa-staging-env"
        PRODUCTION = "rabiaa-prod-env"
        EC2_PRODUCTION_HOST= "184.73.71.185"
        EC2_PRODUCTION_HOST_2= "3.86.223.248"
    }


    agent any

   stages{

        stage ('Build image'){
            agent{label 'test'}
            steps{
                script{
                    sh 'docker build -t ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} .'
                }
            }
        }

        stage ('Run a container and Test Image'){
            agent{label 'test'}
            steps{
                script{
                    sh '''
                       docker stop ${CONTAINER_NAME} || true
                       docker rm ${CONTAINER_NAME} || true
                       docker run -d --name ${CONTAINER_NAME} -e PORT=5000 -p 5000:5000 ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                       sleep 5
                       curl http://localhost:5000 | grep -iq "Hello world!"
                    '''
                }
            }
        }

        stage ('save artifact and clean env'){
            agent{label 'test'}
            environment{
                PASSWORD = credentials('dockerhub_password')
            }
            steps{
                script{
                    sh '''
                       docker login -u ${USERNAME} -p ${PASSWORD}
                       docker push ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                       docker stop ${CONTAINER_NAME}
                       docker rm ${CONTAINER_NAME}
                       docker rmi ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} || true
                    '''
                }
            }
        }

        stage ('deploy app on staging env'){
            agent any
            environment{
                HEROKU_API_KEY = credentials('heroku_apikey')
            }
            steps{
                script{
                    sh '''
                       heroku container:login
                       heroku create $STAGING || echo "project already exist"
                       heroku container:push -a $STAGING web
                       heroku container:release -a $STAGING web
                       
                    '''
                    slackSend (color: '#00F000', message: "TEST: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
                }
            }
        }

        stage ('deploy app on preprod env'){
            agent any
            when {
                expression { GIT_BRANCH == 'origin/master'}
            }
            environment{
                HEROKU_API_KEY = credentials('heroku_apikey')
            }
            steps{
                script{
                    sh '''
                       heroku container:login
                       heroku create $PRODUCTION || echo "project already exist"
                       heroku container:push -a $PRODUCTION web
                       heroku container:release -a $PRODUCTION web
                    '''
                }
            }
        }

        stage ('deploy app on Prod env'){
            agent any
            when {
                expression { GIT_BRANCH == 'origin/master'}
            }
            environment{
                HEROKU_API_KEY = credentials('heroku_apikey')
            }
            steps{
                withCredentials([sshUserPrivateKey(credentialsId: "ec2_private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) {
                    script{
                        timeout(time: 15, unit: "MINUTES") {
                            input message: 'Do you want to approve the deploy in production?', ok: 'Yes'
                        }	
                        sh '''
                           ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker stop ${CONTAINER_NAME} || true
                           ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker rm ${CONTAINER_NAME} || true
                           ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker rmi ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} || true
                           ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker run --name $CONTAINER_NAME -d -e PORT=5000 -p 5000:5000 $USERNAME/$IMAGE_NAME:$IMAGE_TAG 
                           ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST_2} docker stop ${CONTAINER_NAME} || true
                           ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST_2} docker rm ${CONTAINER_NAME} || true
                           ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST_2} docker rmi ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} || true
                           ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST_2} docker run --name $CONTAINER_NAME -d -e PORT=5000 -p 5000:5000 $USERNAME/$IMAGE_NAME:$IMAGE_TAG 

                        '''
                    }
                }
            }
        }

    }
    post {
        always {
            script {
                if ( currentBuild.result == "SUCCESS" ) {
                    slackSend color: "good", message: "CONGRATULATION: Job ${env.JOB_NAME} with buildnumber ${env.BUILD_NUMBER} was successful ! more info ${env.BUILD_URL}"
                }
                else if( currentBuild.result == "FAILURE" ) { 
                    slackSend color: "danger", message: "BAD NEWS:Job ${env.JOB_NAME} with buildnumber ${env.BUILD_NUMBER} was failed ! more info ${env.BUILD_URL}"
                }
                else if( currentBuild.result == "UNSTABLE" ) { 
                    slackSend color: "warning", message: "BAD NEWS:Job ${env.JOB_NAME} with buildnumber ${env.BUILD_NUMBER} was unstable ! more info ${env.BUILD_URL}"
                }
                else {
                    slackSend color: "danger", message: "BAD NEWS:Job ${env.JOB_NAME} with buildnumber ${env.BUILD_NUMBER} its result was unclear ! more info ${env.BUILD_URL}"	
                }
            }
        }
    }
}
