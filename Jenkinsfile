pipeline{
    environment{
        IMAGE_NAME = "alpinehelloworld"
        IMAGE_TAG = "latest"
        USERNAME = "26021973"
        CONTAINTER_NAME = "alpinehelloworld"
        STAGING = "seb-staging-env"
        PRODUCTION = "seb-prod-env"
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
            when {
                expression { GIT_BRANCH == 'origin/master'}
            }
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
        stage ('deploy app on prod env') {
            agent any
            when {
                expression { GIT_BRANCH == 'origin/master'}
            }
            environment {
                HEROKU_API_KEY=credentials('heroku_apikey')
            }
           steps {
               script {
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
        }              
    }
}