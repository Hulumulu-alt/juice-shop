pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        SONAR_TOKEN = credentials('sonarqube-token')
        IMAGE_NAME = 'nazyvaevdocker/juice-shop'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DEFECTDOJO_TOKEN = credentials('defectdojo-token')
        TELEGRAM_TOKEN = credentials('telegram-bot-token')
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
                            echo "[INFO] Установка зависимостей..."
                            npm install --legacy-peer-deps

                            echo "[INFO] Запуск анализа SonarQube..."
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

        stage('Build Juice Shop Image') {
            steps {
                sh '''
                    echo "[INFO] Сборка Docker-образа..."
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
                        echo "[INFO] Обновление Helm values.yaml..."
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

        stage('Deploy to Test') {
            steps {
                sshagent(credentials: ['minikube-ssh']) {
                    sh '''
                        echo "[INFO] Тестовый деплой Juice Shop..."
                        ssh -o StrictHostKeyChecking=no nazyvaev@192.168.56.102 '
                        cd ~/juice-shop/helm &&
                        helm upgrade --install juice-shop . --namespace juice-scan-ns --create-namespace'
                    '''
                }
            }
        }

        stage('OWASP ZAP Scan') {
            steps {
                    sh '''
                        echo "[INFO] Запуск OWASP ZAP..."
                        cd /var/lib/jenkins/zap-scan
                        chmod +x zap-scan.sh
                        ./zap-scan.sh || echo "[WARN] ZAP завершился с ошибкой"
                    '''
                    env.HAS_HIGH_VULNS = readFile('has_high.txt').trim()
            }
        }

        stage('Deploy to Prodaction') {
            when {
                expression { return env.HAS_HIGH_VULNS == 'false' }
            }
            steps {
                sshagent(credentials: ['minikube-ssh']) {
                    sh '''
                        echo "[INFO] Продакшен-деплой Juice Shop..."
                        ssh -o StrictHostKeyChecking=no nazyvaev@192.168.56.102 '
                        cd ~/juice-shop/helm &&
                        helm upgrade --install juice-shop . --namespace default --create-namespace'
                    '''
                }
            }
        }

        stage('Archive ZAP Reports') {
            steps {
                archiveArtifacts artifacts: 'zap-report/zap_report.html', allowEmptyArchive: true
                archiveArtifacts artifacts: 'zap-report/zap-report.xml', allowEmptyArchive: true
                archiveArtifacts artifacts: 'zap-report/zap-report.json', allowEmptyArchive: true
            }
        }
    }
}
