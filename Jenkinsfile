pipeline {
     environment {
       IMAGE_NAME = "serviio"
       IMAGE_NAME_2 = "Serviio"
       IMAGE_TAG = "latest"
       STAGING = "serviio-staging"
       PRODUCTION = "serviio-production"
 
       
     }
     agent any
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
                    docker run --name $IMAGE_NAME_2 -d --net host blackibanez/$IMAGE_NAME:$IMAGE_TAG
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
                   input message: 'Do you want to approve the deploy on Diskstation ?', ok: 'Yes'                            
              }
            sh '''
              ssh -o StrictHostKeyChecking=no  -p ${PORT_SSH} ${NUSER}@${PRODUCTION_IP_HOST} mkdir -p ${SERVIIO_VOLUME}/log ${SERVIIO_VOLUME}/plugins ${SERVIIO_VOLUME}/library
              ssh -o StrictHostKeyChecking=no  -p ${PORT_SSH} ${NUSER}@${PRODUCTION_IP_HOST} /usr/local/bin/docker pull $IMAGE_NAME_2 || true
              ssh -o StrictHostKeyChecking=no  -p ${PORT_SSH} ${NUSER}@${PRODUCTION_IP_HOST} /usr/local/bin/docker stop $IMAGE_NAME_2 || true
              ssh -o StrictHostKeyChecking=no  -p ${PORT_SSH} ${NUSER}@${PRODUCTION_IP_HOST} /usr/local/bin/docker rm $IMAGE_NAME_2  || true
              ssh -o StrictHostKeyChecking=no  -p ${PORT_SSH} ${NUSER}@${PRODUCTION_IP_HOST} /usr/local/bin/docker run --name $IMAGE_NAME_2 -d --net host -v ${SERVIIO_VOLUME}/log:/opt/serviio/log  -v ${SERVIIO_VOLUME}/library:/opt/serviio/library -v ${SERVIIO_VOLUME}/plugins:/opt/serviio/plugins -v $MEDIA_VOLUME:/media/serviio/adult -v $DOWNLOADS_VOLUME:/media/serviio/downloads  blackibanez/$IMAGE_NAME:$IMAGE_TAG  || true
            '''
             }
           }
        }
     }
    }      
 }
}
