pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        SONAR_TOKEN = credentials('sonarqube-token')
        IMAGE_NAME = 'nazyvaevdocker/juice-shop'
        IMAGE_TAG = "${BUILD_NUMBER}"
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
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                        set -e
                        echo "[INFO] Installing dependencies with legacy peer deps"
                        npm install --legacy-peer-deps || true

                        echo "[INFO] Starting SonarQube scan..."
                        npx sonar-scanner \
                          -Dsonar.projectKey=juice-shop \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://localhost:9000 \
                          -Dsonar.login=$SONAR_TOKEN || echo "[WARN] SonarQube scan failed but continuing"
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "[INFO] Building Docker image..."
                    docker build -t $IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Push Docker Image') {
            environment {
                DOCKER_CREDENTIALS = credentials('dockerhub-credentials')
            }
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        docker.image("$IMAGE_NAME:$IMAGE_TAG").push()
                    }
                }
            }
        }

        stage('Update Helm values') {
            environment {
                GIT_REPO_NAME = "juice-shop"
                GIT_USER_NAME = "Hulumulu-alt"
            }
            steps {
                withCredentials([string(credentialsId: 'github-credentials-juice', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        echo "[INFO] Configuring git and updating Helm values..."
                        git config --global user.email "nazivaevaleksey8983@gmail.com"
                        git config --global user.name "Hulumulu-alt"

                        if [ -f helm/values.yaml ]; then
                            sed -i "s/replaceJuiceTag/${BUILD_NUMBER}/g" helm/values.yaml
                            git add helm/values.yaml
                            git commit -m "Update Juice Shop tag to ${BUILD_NUMBER}" || true
                            git push https://x-access-token:$GITHUB_TOKEN@github.com/Hulumulu-alt/juice-shop.git HEAD:master
                        else
                            echo "[ERROR] helm/values.yaml not found, skipping commit step"
                        fi
                    '''
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                sshagent(credentials: ['minikube-ssh']) {
                    sh '''
                        echo "[INFO] Deploying to Minikube with Helm..."
                        ssh -o StrictHostKeyChecking=no nazyvaev@192.168.56.102 '
                        cd ~/juice-shop/helm &&
                        helm upgrade --install juice-shop . --namespace default'
                    '''
                }
            }
        }
    }
}
