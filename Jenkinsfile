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
        stage ('Checkov Scan') {
            steps {
                script {
                    sh '''
                    cd helmcharts
                    sudo checkov -d .
                    '''
                }
            }
        }
	/*
        stage('ZAP Integration') {
            steps {
                script {
                    // Create directories to store ZAP reports and config
                    sh 'mkdir -p ${WORK_DIR}'
                    sh 'mkdir -p ${CONFIG_DIR}'
                    // Optionally, copy the alert filter file to ${CONFIG_DIR} if not already present
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

        stage('Start OWASP ZAP Docker Container') {
            steps {
                script {
                    // Start OWASP ZAP Docker container
                    sh ''' 
                    sudo docker run -d --name zap -p 8081:8081 ${ZAP_IMAGE} zap.sh -daemon -port 8081 -host 0.0.0.0 
                    sleep 30 
                    ''' 
                }
            }
        }

        stage('Run OWASP ZAP Scan') {
            steps {
                script {
                    // Run ZAP API scan with the alert filter file
                    def zapScanResult = sh script: """ 
                    sudo docker run --rm --net='host' -v \$(pwd)/${WORK_DIR}:/zap/wrk -v \$(pwd)/${CONFIG_DIR}:/zap/config ${ZAP_IMAGE} zap-api-scan.py -t ${TARGET_URL} -f openapi -r /zap/wrk/zap_report.html -J /zap/config/alert_filters.json 
                    """, returnStatus: true

                    // Check the exit code and print warnings or errors
                    if (zapScanResult != 0) {
                        echo "Error running OWASP ZAP scan: script returned exit code ${zapScanResult}"
                    }
                }
            }
        }

        stage('Generate and Archive Report') {
            steps {
                script {
                    // Archive ZAP report as a build artifact
                    archiveArtifacts artifacts: 'zap_work/zap_report.html', onlyIfSuccessful: true
                }
            }
        }

        stage('Stop OWASP ZAP Docker Container') {
            steps {
                script {
                    // Stop OWASP ZAP Docker container
                    sh "sudo docker stop zap && sudo docker rm zap"
                }
            }
        }
	 */   
        stage('Trivy Scan') {
            steps {
                sh 'trivy image $IMAGE_REPO:$IMAGE_VERSION'
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
