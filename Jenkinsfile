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
                    runCmd('docker build -t hadilsamoud/mern-server:latest ./Server')
                    runCmd('docker build -t hadilsamoud/mern-client:latest ./Client')
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
                            sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
                        } else {
                            bat 'echo %DH_PASS%| docker login -u %DH_USER% --password-stdin'
                        }
                        runCmd('docker push hadilsamoud/mern-server:latest')
                        runCmd('docker push hadilsamoud/mern-client:latest')
                    }
                }
            }
        }

        stage('Deploiement') {
            steps {
                script {
                    runCmd('docker compose down')
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
