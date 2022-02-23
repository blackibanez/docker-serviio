pipeline {
     environment {
       IMAGE_NAME = "serviio"
       IMAGE_NAME_2 = "Serviio"
       IMAGE_TAG = "latest"
       STAGING = "serviio-staging"
       PRODUCTION = "serviio-production"
 
       
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
                    sleep 90
                 '''
               }
            }
       }
       stage('Test image') {
           agent any
           steps {
              script {
                sh '''
                    curl http://localhost:23423/console/#/app/welcome/ | grep "serviio"
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
     stage('Push image in production and deploy it') {
       when {
              expression { GIT_BRANCH == 'origin/master' }
            }
      agent any
      environment {
           PRODUCTION_IP_HOST = credentials('diskstation_2_ip')
           SERVIIO_VOLUME = credentials('serviio_volume_folder')
           MEDIA_VOLUME = credentials('media_volume_folder')
           DOWNLOADS_VOLUME = credentials('downloads_volume_folder')
           PORT_SSH = credentials('port_ssh')
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
                   input message: 'Do you want to approve the deploy on $PRODUCTION_IP_HOST?', ok: 'Yes'                            
              }
            sh '''
              ssh -o StrictHostKeyChecking=no  -p ${PORT_SSH} ${NUSER}@${PRODUCTION_IP_HOST} /usr/local/bin/docker stop $IMAGE_NAME_2 || true
              ssh -o StrictHostKeyChecking=no  -p ${PORT_SSH} ${NUSER}@${PRODUCTION_IP_HOST} /usr/local/bin/docker rm $IMAGE_NAME_2  || true
              ssh -o StrictHostKeyChecking=no  -p ${PORT_SSH} ${NUSER}@${PRODUCTION_IP_HOST} /usr/local/bin/docker rmi blackibanez/$IMAGE_NAME:$IMAGE_TAG  || true
              ssh -o StrictHostKeyChecking=no  -p ${PORT_SSH} ${NUSER}@${PRODUCTION_IP_HOST} /usr/local/bin/docker run --name $IMAGE_NAME_2 -d -p 23423:23423 -p 8895:8895 -p 1900:1900  -v $MEDIA_VOLUME:/media/serviio/adult -v $DOWNLOADS_VOLUME:/media/serviio/downloads  blackibanez/$IMAGE_NAME:$IMAGE_TAG  || true
            '''
             }
           }
        }
     }
    }      
 }
}
