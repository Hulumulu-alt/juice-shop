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
                    echo "[INFO] Сборка Docker-образа Juice Shop..."
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
                        echo "[INFO] Обновление тега в Helm values.yaml..."
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
                        echo "[INFO] Деплой для тестов OWASP ZAP"
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
            echo "[INFO] Запуск сканирования OWASP ZAP (локально)..."
            cd /var/lib/jenkins/zap-scan
            chmod +x zap-scan.sh
            ./zap-scan.sh || echo "[WARN] ZAP завершился с ошибкой или нашёл уязвимости"

            echo "[INFO] Копирование ZAP-отчётов в рабочую директорию Jenkins..."
            mkdir -p ${WORKSPACE}/zap-report
            cp -v reports/zap_report.html ${WORKSPACE}/zap-report/ || true
            cp -v reports/zap-report.xml ${WORKSPACE}/zap-report/ || true
            cp -v reports/zap-report.json ${WORKSPACE}/zap-report/ || true

            echo "[INFO] Проверка на уязвимости уровня High в XML-отчёте..."
            HIGH_COUNT=$(xmllint --xpath "count(//alertitem[riskdesc='High (Medium)'])" ${WORKSPACE}/zap-report/zap-report.xml || echo 0)

            echo "[INFO] Найдено уязвимостей уровня High: $HIGH_COUNT"

            if [ "$HIGH_COUNT" -gt 0 ]; then
                echo "[ALERT] Обнаружены уязвимости уровня High. Прерываем пайплайн..."
                curl -s -X POST https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage \
                    -d chat_id=$TELEGRAM_CHAT_ID \
                    -d text="❌ OWASP ZAP обнаружил $HIGH_COUNT уязвимост(ей) уровня High. Развёртывание отменено."
                exit 1
            else
                echo "[SUCCESS] Сканирование завершено успешно. High-уязвимости не найдены."
                curl -s -X POST https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage \
                    -d chat_id=$TELEGRAM_CHAT_ID \
                    -d text="✅ Сканирование OWASP ZAP завершено. Уязвимости уровня High не обнаружены."

                echo "[INFO] Удаление временного namespace juice-scan-ns..."
                ssh -o StrictHostKeyChecking=no nazyvaev@192.168.56.102 '
                    kubectl delete namespace juice-scan-ns --ignore-not-found
                '
            fi
        '''
    }
}

 stage('Deploy to Minikube') {
            steps {
                sshagent(credentials: ['minikube-ssh']) {
                    sh '''
                        echo "[INFO] Деплой в кластер Minikube..."
                        ssh -o StrictHostKeyChecking=no nazyvaev@192.168.56.102 '
                        cd ~/juice-shop/helm &&
                        helm upgrade --install juice-shop . --namespace default --create-namespace'
                    '''
                }
            }
        }
        
        stage('Upload to DefectDojo') {
            steps {
                sh '''
                    echo "[INFO] Отправка отчета OWASP ZAP в DefectDojo..."
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

        stage('Archive ZAP Reports') {
            steps {
                archiveArtifacts artifacts: 'zap-report/zap_report.html', allowEmptyArchive: true
                archiveArtifacts artifacts: 'zap-report/zap-report.xml', allowEmptyArchive: true
                archiveArtifacts artifacts: 'zap-report/zap-report.json', allowEmptyArchive: true
            }
        }
    }
}
