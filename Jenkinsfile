// Lance une commande avec sh (Linux) ou bat (Windows)
def runCmd(String cmd) {
    if (isUnix()) { sh cmd } else { bat cmd }
}

pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build des images') {
            steps {
                script {
                    runCmd('docker build -t mern-server ./Server')
                    runCmd('docker build -t mern-client ./Client')
                }
            }
        }

        stage('Push sur Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    script {
                        if (isUnix()) {
                            sh '''
                                echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                                docker tag mern-server "$DH_USER/mern-server:latest"
                                docker tag mern-client "$DH_USER/mern-client:latest"
                                docker push "$DH_USER/mern-server:latest"
                                docker push "$DH_USER/mern-client:latest"
                            '''
                        } else {
                            bat '''
                                echo %DH_PASS%| docker login -u %DH_USER% --password-stdin
                                docker tag mern-server %DH_USER%/mern-server:latest
                                docker tag mern-client %DH_USER%/mern-client:latest
                                docker push %DH_USER%/mern-server:latest
                                docker push %DH_USER%/mern-client:latest
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploiement') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'docker rm -f mongodb server client || true'
                    } else {
                        bat 'docker rm -f mongodb server client || exit /b 0'
                    }
                    runCmd('docker compose up -d')
                }
            }
        }
    }

    post {
        always {
            script { runCmd('docker logout') }
        }
    }
}
