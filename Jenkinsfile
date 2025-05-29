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
                        helm upgrade --install juice-shop . \
                        --namespace juice-scan-ns --create-namespace \
                        --set ingress.hosts[0].host=juice-scan.local
                    '''
                }
            }
        }

        stage('OWASP ZAP Scan') {
    steps {
        script {
            sh '''
                echo "[INFO] Запуск OWASP ZAP..."
                cd /var/lib/jenkins/zap-scan
                chmod +x zap-scan.sh
                ./zap-scan.sh || echo "[WARN] ZAP завершился с ошибкой"

                echo "[INFO] Копирование ZAP-отчётов в рабочую директорию Jenkins..."
                mkdir -p ${WORKSPACE}/zap-report
                cp -v reports/zap_report.html ${WORKSPACE}/zap-report/ || true
                cp -v reports/zap-report.xml ${WORKSPACE}/zap-report/ || true
                cp -v reports/zap-report.json ${WORKSPACE}/zap-report/ || true

                echo "[INFO] Анализ отчёта на High-уязвимости..."
                HIGH_COUNT=$(xmllint --xpath "count(//alertitem[riskdesc='Critical (High)'])" ${WORKSPACE}/zap-report/zap-report.xml || echo 0)

                echo "[INFO] Найдено уязвимостей уровня High: $HIGH_COUNT"

                if [ "$HIGH_COUNT" -gt 0 ]; then
                    echo "true" > ${WORKSPACE}/has_high.txt
                    echo "[ALERT] Найдены High-уязвимости: $HIGH_COUNT"
                    curl -s -X POST https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage \
                        -d chat_id=$TELEGRAM_CHAT_ID \
                        -d text="❌ OWASP ZAP обнаружил $HIGH_COUNT High-уязвимост(ей). Развёртывание отменено."
                else
                    echo "false" > ${WORKSPACE}/has_high.txt
                    echo "[SUCCESS] High-уязвимости не обнаружены."
                    curl -s -X POST https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage \
                        -d chat_id=$TELEGRAM_CHAT_ID \
                        -d text="✅ OWASP ZAP завершён. High-уязвимостей не найдено."
                fi
            '''
        }
    }
       
            post {
                always {
                    sshagent(credentials: ['minikube-ssh']) {
                        sh '''
                            echo "[INFO] Удаление juice-scan-ns..."
                            ssh -o StrictHostKeyChecking=no nazyvaev@192.168.56.102 '
                                kubectl delete namespace juice-scan-ns --ignore-not-found'
                        '''    
                    }
                    sh '''
            echo "[INFO] Загрузка отчета OWASP ZAP в DefectDojo..."
            if [ -f zap-report/zap-report.xml ]; then
                RESPONSE=$(curl -s -w "%{http_code}" -o /tmp/dd_response.txt -X POST http://localhost:8085/api/v2/import-scan/ \
                    -H "Authorization: Token $DEFECTDOJO_TOKEN" \
                    -F "scan_type=ZAP Scan" \
                    -F "minimum_severity=Low" \
                    -F "engagement=1" \
                    -F "lead=1" \
                    -F "file=@zap-report/zap-report.xml")

                if [ "$RESPONSE" -eq 201 ]; then
                    echo "[SUCCESS] Отчёт успешно загружен в DefectDojo."
                    curl -s -X POST https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage \
                      -d chat_id=$TELEGRAM_CHAT_ID \
                      -d text="📤 OWASP ZAP отчёт успешно загружен в DefectDojo."
                else
                    echo "[ERROR] Не удалось загрузить отчёт в DefectDojo. HTTP $RESPONSE"
                    ERROR_MSG=$(cat /tmp/dd_response.txt)
                    curl -s -X POST https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage \
                      -d chat_id=$TELEGRAM_CHAT_ID \
                      -d text="⚠️ Ошибка загрузки OWASP ZAP отчёта в DefectDojo: HTTP $RESPONSE. Ответ: $ERROR_MSG"
                fi
            else
                echo "[WARN] Файл zap-report.xml не найден. Пропуск загрузки."
                curl -s -X POST https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage \
                  -d chat_id=$TELEGRAM_CHAT_ID \
                  -d text="⚠️ ZAP отчёт не найден. Загрузка в DefectDojo пропущена."
            fi
        '''
         }
       }   
    }
        
        stage('Deploy to Minikube') {
            steps {
                script {
                    def hasHigh = readFile('has_high.txt').trim()
                    if (hasHigh == 'false') {
                        sshagent(credentials: ['minikube-ssh']) {
                            sh '''
                                echo "[INFO] Продакшен-деплой Juice Shop..."
                                ssh -o StrictHostKeyChecking=no nazyvaev@192.168.56.102 '
                                cd ~/juice-shop/helm &&
                                helm upgrade --install juice-shop . \
                                --namespace default \
                                --set ingress.hosts[0].host=juice-shop.local
                            '''
                        }
                        sh '''
                            echo "[INFO] Отправка Telegram-уведомления о продакшен-деплое..."
                            curl -s -X POST https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage \
                                -d chat_id=$TELEGRAM_CHAT_ID \
                                -d text="🚀 Juice Shop успешно развёрнут в production."
                        '''
                    } else {
                        echo "[INFO] Продакшен-деплой пропущен из-за найденных High-уязвимостей."
                    }
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
