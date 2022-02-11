
/* import library */ 
// @Library('slack_Notifier')_

pipeline{
    environment{
        IMAGE_NAME = "alpinehelloworld"
        IMAGE_TAG = "latest"
        USERNAME = "frabiaa"
        CONTAINTER_NAME = "alpinehelloworld"
        STAGING = "rabiaa-staging-env"
        PRODUCTION = "rabiaa-prod-env"
        EC2_PRODUCTION_HOST= "184.73.71.185"
    }


    agent any

    stages {

        stage ('build mon image') {
           steps {
               script {
                   sh 'docker build -t ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} .'
               }
           } 
        }

        stage ('Run a contrainer and test image') {
           steps {
               script {
                   sh '''
                    docker stop ${CONTAINTER_NAME} || true
                    docker rm ${CONTAINTER_NAME} || true
                    docker run -d --name ${CONTAINTER_NAME} -e PORT=5000 -p 5000:5000 ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 5
                    curl http://localhost:5000 | grep -iq "Hello world!"
                   '''
               }
           } 
        }

        stage ('Save artifact and clean env') {
            agent any
            environment {
                PASSWORD=credentials('dockerhub_password')
            }
           steps {
               script {
                   sh '''
                    docker login -u ${USERNAME} -p ${PASSWORD}
                    docker push ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker stop ${CONTAINTER_NAME}
                    docker rm ${CONTAINTER_NAME}
                    docker rmi ${USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                   '''
               }
           } 
        }
        stage ('deploy app on staging env') {
            agent any
           
            environment {
                HEROKU_API_KEY=credentials('heroku_apikey')
            }
           steps {
               script {
                   sh '''
                        heroku container:login
                        heroku create $STAGING || echo "project already exist"
                        heroku container:push -a $STAGING web
                        heroku container:release -a $STAGING web
                   '''
               }
           } 
        } 
        stage ('deploy app on pre prod env') {
            agent any
           
            environment {
                HEROKU_API_KEY=credentials('heroku_apikey')
            }
           steps {
               script {
                    slackSend color: "warning", message: "VALIDATION REQUIRED ${env.BUILD_URL}: Job ${env.JOB_NAME} with buildnumber ${env.BUILD_NUMBER} for testing  http://${env.STAGING}.herokuapp.com"
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to approve the deploy in production?', ok: 'Yes'
                    }
                   sh '''
                        heroku container:login
                        heroku create $PRODUCTION || echo "project already exist"
                        heroku container:push -a $PRODUCTION web
                        heroku container:release -a $PRODUCTION web
                   '''
               }
           } 
            
            stage ('deploy app on prod env') {
            agent any
            when {
                expression { GIT_BRANCH == 'origin/master'}
            }
            environment {
                HEROKU_API_KEY=credentials('heroku_apikey')
            }
           steps {
              withCredentials([sshUserPrivateKey(credentialsId: "ec2_private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) {
                    script{
                        timeout(time: 15, unit: "MINUTES") {
                            input message: 'Do you want to approve the deploy in production?', ok: 'Yes'
                        }	
                        sh '''
                           ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker run --name $CONTAINER_NAME -d -e PORT=5000 -p 5000:5000 $USERNAME/$IMAGE_NAME:$IMAGE_TAG 
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
