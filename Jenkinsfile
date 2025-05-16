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
                    sh """
                        npm install
                        npx sonar-scanner \
                          -Dsonar.projectKey=juice-shop \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://localhost:9000 \
                          -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
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
                withCredentials([string(credentialsId: 'github-credentials', variable: 'GITHUB_TOKEN')]) {
                    sh """
                        git config --global user.email "nazivaevaleksey8983@gmail.com"
                        git config --global user.name "Hulumulu-alt"
                        sed -i "s/replaceJuiceTag/${BUILD_NUMBER}/g" helm/values.yaml || true
                        git add helm/values.yaml
                        git commit -m "Update Juice Shop tag to ${BUILD_NUMBER}" || true
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:master
                    """
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                sshagent(credentials: ['minikube-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no nazyvaev@192.168.56.102 '
                        cd ~/juice-shop/helm &&
                        helm upgrade --install juice-shop . --namespace default'
                    '''
                }
            }
        }
    }
}
