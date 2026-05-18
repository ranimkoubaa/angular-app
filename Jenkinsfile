pipeline {
    agent any
    stages {
        stage('Clone Stage') {
            steps {
                git 'https://github.com/ranimkoubaa/angular-app.git'
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    DOCKER_TAG = bat(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    DOCKER_TAG = DOCKER_TAG.readLines().last()
                }
                bat "docker build -t ranimkoubaa/angular-app:${DOCKER_TAG} ."
            }
        }
        stage('DockerHub Push') {
            steps {
                withCredentials([string(credentialsId: 'mydockerhubpassword', variable: 'DockerHubPassword')]) {
                    bat "docker login -u ranimkoubaa -p %DockerHubPassword%"
                }
                bat "docker push ranimkoubaa/angular-app:${DOCKER_TAG}"
            }
        }
        stage('Deploy') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'Vagrant_ssh', keyFileVariable: 'SSH_KEY')]) {
                    bat """
                        icacls %SSH_KEY% /inheritance:r
                        icacls %SSH_KEY% /grant:r "%USERNAME%:R"
                        ssh -i %SSH_KEY% -o StrictHostKeyChecking=no jenkins@192.168.56.103 "sudo docker pull ranimkoubaa/angular-app:${DOCKER_TAG} && sudo docker run -d -p 80:80 ranimkoubaa/angular-app:${DOCKER_TAG}"
                    """
                }
            }
        }
    }
}