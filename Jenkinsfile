pipeline {
    agent any
    environment {
        GITHUB_CREDS=credentials('swetha-github-creds')
        IMAGE_NAME= 'my-final-image'
        IMAGE_REPO='sriswetha06/my-final-image'
        IMAGE_VERSION='v3'
        DOCKERHUB_CREDS=credentials('swetha-docker-creds')
        COSIGN_PASSWORD=credentials('cosign-password')
        COSIGN_PRIVATE_KEY=credentials('cosign-private-key')
        COSIGN_PUBLIC_KEY=credentials('cosign-public-key')
         ZAP_IMAGE = 'ghcr.io/zaproxy/zaproxy:stable' 
         TARGET_URL = 'http://10.45.88.84:5000' 
         WORK_DIR = 'zap_work' 
         CONFIG_DIR = 'zap_config'  
    }
    stages {     
    stage('Pre SAST') {
             steps {
                 sh 'gitleaks version'
                 sh 'gitleaks detect --source . -v || true'
             }
         }
          stage('Bandit Scan') {
            steps {
                script {
                   sh 'echo "yes" | sudo apt install python3-bandit'
                   sh 'bandit --version'
                   sh  'bandit -r . || true'
                }
            }
          }
        stage('Build Docker Image') {
            steps {
                script {
                   sh 'sudo docker build -t $IMAGE_NAME . '
                }
            }
        }
           stage('Docker Login'){
            steps{
                withCredentials([usernamePassword(credentialsId: 'swetha-docker-creds', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                      sh "docker login -u $USERNAME -p $PASSWORD"
                }
            }
        }
        stage ('Tag and Push') {
            steps {
                script {
                    sh 'sudo docker tag $IMAGE_NAME $IMAGE_REPO:$IMAGE_VERSION'
                    sh 'sudo docker push $IMAGE_REPO:$IMAGE_VERSION'
                }
            }
        }
        stage('Trivy Scan') {
            steps {
                sh 'trivy image $IMAGE_REPO:$IMAGE_VERSION'
            }
        }
        stage('Zap Scan') {
              steps {
                  sh 'mkdir -p ${WORK_DIR}' 
                  sh 'mkdir -p ${CONFIG_DIR}'
                  writeFile file: "${CONFIG_DIR}/alert_filters.json", text: ''' 
                    [ 
                        {"ruleId": "100001", "url": ".*", "newLevel": 0, "urlIsRegex": true}, 

                        {"ruleId": "10020", "url": ".*", "newLevel": 0, "urlIsRegex": true}, 

                        {"ruleId": "10021", "url": ".*", "newLevel": 0, "urlIsRegex": true}, 

                        {"ruleId": "10036", "url": ".*", "newLevel": 0, "urlIsRegex": true}, 

                        {"ruleId": "10038", "url": ".*", "newLevel": 0, "urlIsRegex": true}, 

                        {"ruleId": "10063", "url": ".*", "newLevel": 0, "urlIsRegex": true} 

                    ] 
                    '''
                }
            }
        }

	stage('Sign and Verify image with Cosign'){
            steps{
                sh 'echo "y" | cosign sign --key $COSIGN_PRIVATE_KEY docker.io/$IMAGE_REPO:$IMAGE_VERSION'
                sh 'cosign verify --key $COSIGN_PUBLIC_KEY docker.io/$IMAGE_REPO:$IMAGE_VERSION'
                echo 'Image signed successfully'
            }
        }
    }
}
