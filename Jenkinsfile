pipeline {
     environment {
       IMAGE_NAME = "serviio"
       IMAGE_NAME_2 = "serviio_test"
       IMAGE_TAG = "latest"
       STAGING = "serviio-staging"
       PRODUCTION = "serviio-production"
       PRODUCTION_IP_HOST = "52.23.205.35"
       
     }
     agent none
     stages {
         stage('Build image') {
             agent any
             steps {
                script {
                  sh 'docker build -t blackibanez/$IMAGE_NAME:$IMAGE_TAG .'
                }
             }
        }
        stage('Run container based on builded image') {
            agent any
            steps {
                 script {
                 sh '''
                    docker stop $IMAGE_NAME_2 || true
                    docker rm $IMAGE_NAME_2 || true
                    docker run --name $IMAGE_NAME_2 -d -p 23423:23423 -p 8895:8895 -p 1900:1900 blackibanez/$IMAGE_NAME:$IMAGE_TAG
                    sleep 180
                 '''
               }
            }
       }
       stage('Test image') {
           agent any
           steps {
              script {
                sh '''
                    curl http://172.17.0.1:23423/console/#/app/welcome/ | grep "serviio"
                '''
              }
           }
      }
       stage('Push on Dockerhub') {
          agent any
          environment {
               PASSWORD = credentials('password_dockerhub')
          } 
          steps {
             script {
               sh '''
                 docker login -u blackibanez -p $PASSWORD
                 docker push blackibanez/$IMAGE_NAME:$IMAGE_TAG
               '''
             }
          }
     } 
      stage('Clean Container') {
          agent any
          steps {
             script {
               sh '''
                 docker stop $IMAGE_NAME_2
                 docker rm $IMAGE_NAME_2
               '''
             }
          }
     }
     stage('Push image in staging and deploy it') {
       when {
              expression { GIT_BRANCH == 'origin/master' }
            }
      agent any
      environment {
          HEROKU_API_KEY = credentials('heroku_api_key')
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
     stage('Push image in production and deploy it') {
       when {
              expression { GIT_BRANCH == 'origin/master' }
            }
      agent any
      environment {
          HEROKU_API_KEY = credentials('heroku_api_key')
      }  
      steps {
           withCredentials([sshUserPrivateKey(credentialsId: "private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) 
           {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') 
            {                        
                 script
             {                           
                  timeout(time: 15, unit: "MINUTES") 
              {                                
                   input message: 'Do you want to approve the deploy in production?', ok: 'Yes'                            
              }
            sh '''
              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker stop $IMAGE_NAME  || true
              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker rm $IMAGE_NAME  || true
              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker rmi blackibanez/$IMAGE_NAME:$IMAGE_TAG  || true
              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker run --name $IMAGE_NAME_2 -d -p 23423:23423 -p 8895:8895 -p 1900:1900 blackibanez/$IMAGE_NAME:$IMAGE_TAG  || true
            '''
             }
           }
        }
     }
    }      
  }
}
