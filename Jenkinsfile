pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        SONAR_TOKEN = credentials('sonarqube-token')
        IMAGE_NAME = 'nazyvaevdocker/juice-shop'
        IMAGE_TAG = "${BUILD_NUMBER}"
        TELEGRAM_TOKEN = credentials('telegram-token')
        TELEGRAM_CHAT_ID = credentials('telegram-chat-id')
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir()
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv("${SONARQUBE_SERVER}") {
                        sh '''
                            npm install --legacy-peer-deps
                            npx sonar-scanner \
                              -Dsonar.projectKey=juice-shop \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=http://localhost:9000 \
                              -Dsonar.login=${SONAR_TOKEN}
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Push Docker Image') {
            environment {
                REGISTRY_CREDENTIALS = credentials('dockerhub-credentials')
            }
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "dockerhub-credentials") {
                        sh '''
                            docker push $IMAGE_NAME:$IMAGE_TAG
                            docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
                            docker push $IMAGE_NAME:latest
                        '''
                    }
                }
            }
        }

        stage('Update Helm Tag') {
            environment {
                GIT_REPO_NAME = "juice-shop"
                GIT_USER_NAME = "Hulumulu-alt"
            }
            steps {
                withCredentials([string(credentialsId: 'github-credentials-juice', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config user.email "nazivaevaleksey8983@gmail.com"
                        git config user.name "Hulumulu-alt"
                        sed -i "s/replaceJuiceTag/${BUILD_NUMBER}/g" helm/values.yaml || true
                        git add helm/values.yaml
                        git commit -m "Обновление тега Docker-образа на ${BUILD_NUMBER}" || true
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:master
                    '''
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sshagent(credentials: ['minikube-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no nazyvaev@192.168.56.102 '
                        cd ~/juice-shop/helm &&
                        helm upgrade --install juice-shop . --namespace default --create-namespace'
                    '''
                }
            }
        }

        stage('OWASP ZAP Scan') {
            steps {
                sh '''
                    cd /var/lib/jenkins/zap-scan
                    chmod +x zap-scan.sh
                    ./zap-scan.sh || true
                    mkdir -p ${WORKSPACE}/zap-report
                    cp -v reports/zap-report.xml ${WORKSPACE}/zap-report/ || true
                '''
            }
        }

        stage('Analyze ZAP Report') {
            steps {
                script {
                    def zapXml = readFile(file: 'zap-report/zap-report.xml')
                    if (zapXml.contains('riskcode="3"')) {
                        sh '''
                            curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage \
                            -d chat_id=${TELEGRAM_CHAT_ID} \
                            -d text="‼ Найдены уязвимости HIGH уровня в Juice Shop. Деплой остановлен."
                        '''
                        error("HIGH-уязвимости найдены. Прерывание пайплайна.")
                    }
                }
            }
        }

        stage('Upload to DefectDojo') {
            environment {
                DEFECTDOJO_TOKEN = credentials('defectdojo-token')
            }
            steps {
                sh '''
                    curl -X POST http://localhost:8085/api/v2/import-scan/ \
                      -H "Authorization: Token $DEFECTDOJO_TOKEN" \
                      -F "scan_type=ZAP Scan" \
                      -F "minimum_severity=Low" \
                      -F "engagement=1" \
                      -F "lead=1" \
                      -F "file=@zap-report/zap-report.xml"
                '''
            }
        }
    }

    post {
    always {
        node {
            echo '[INFO] Архивация ZAP-отчетов...'
            archiveArtifacts artifacts: 'zap-report/zap_report.html', allowEmptyArchive: true
            archiveArtifacts artifacts: 'zap-report/zap-report.xml', allowEmptyArchive: true
            archiveArtifacts artifacts: 'zap-report/zap-report.json', allowEmptyArchive: true
         }
      }
   }
}
