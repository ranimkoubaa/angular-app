def getDockerTag() {
    def rawOutput = bat(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
    def lines = rawOutput.split(/[\r\n]+/)
    def tag = lines[lines.length - 1].trim()
    echo "Raw output: '${rawOutput}'"
    echo "Extracted tag: '${tag}'"
    return tag
}

pipeline {
    agent any

    environment {
        DOCKER_TAG = ''
        DOCKER_USER = 'ranimkoubaa'
        REMOTE_HOST = 'jenkins@192.168.56.103'
        APP_NAME = 'angular-app'
    }

    stages {

        stage('Docker Build') {
            steps {
                script {
                    env.DOCKER_TAG = getDockerTag()
                    echo "==> Building image: ${env.DOCKER_USER}/${env.APP_NAME}:${env.DOCKER_TAG}"
                }
                bat "docker build -t ${env.DOCKER_USER}/${env.APP_NAME}:${env.DOCKER_TAG} ."
            }
        }

        stage('DockerHub Push') {
            steps {
                withCredentials([string(credentialsId: 'mydockerhubpassword', variable: 'DockerHubPassword')]) {
                    bat "docker login -u ${env.DOCKER_USER} -p %DockerHubPassword%"
                }
                bat "docker push ${env.DOCKER_USER}/${env.APP_NAME}:${env.DOCKER_TAG}"
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def tmpKey = "C:\\Windows\\Temp\\jenkins_ssh_${env.BUILD_NUMBER}.pem"

                    withCredentials([sshUserPrivateKey(credentialsId: 'Vagrant_ssh', keyFileVariable: 'SSH_KEY')]) {
                        try {
                            bat """
                                copy "%SSH_KEY%" "${tmpKey}"
                                icacls "${tmpKey}" /inheritance:r
                                icacls "${tmpKey}" /grant:r "%USERNAME%:F"
                            """

                            bat """
                                ssh -i "${tmpKey}" ^
                                    -o StrictHostKeyChecking=no ^
                                    ${env.REMOTE_HOST} ^
                                    "sudo docker ps -q --filter publish=80 | xargs -r sudo docker stop ; sudo docker ps -aq --filter publish=80 | xargs -r sudo docker rm ; sudo docker pull ${env.DOCKER_USER}/${env.APP_NAME}:${env.DOCKER_TAG} && sudo docker run -d --name ${env.APP_NAME} -p 80:80 ${env.DOCKER_USER}/${env.APP_NAME}:${env.DOCKER_TAG}"
                            """

                        } finally {
                            bat "del /f /q \"${tmpKey}\" 2>nul"
                            bat "icacls \"%SSH_KEY%\" /grant:r \"%USERNAME%:F\" 2>nul"
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline réussi !"
            echo "   Image: ${env.DOCKER_USER}/${env.APP_NAME}:${env.DOCKER_TAG}"
            echo "   URL: http://192.168.56.103"
        }
        failure {
            echo "❌ Pipeline échoué. Vérifier les logs ci-dessus."
        }
    }
}