def getDockerTag() {
    def rawOutput = bat(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
    return rawOutput.tokenize('\n').last().trim()
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
                    // Copier la clé dans un fichier temporaire qu'ON contrôle
                    def tmpKey = "C:\\Windows\\Temp\\jenkins_ssh_${env.BUILD_NUMBER}.pem"

                    withCredentials([sshUserPrivateKey(credentialsId: 'Vagrant_ssh', keyFileVariable: 'SSH_KEY')]) {
                        try {
                            // Copier + fixer les permissions
                            bat """
                                copy "%SSH_KEY%" "${tmpKey}"
                                icacls "${tmpKey}" /inheritance:r
                                icacls "${tmpKey}" /grant:r "%USERNAME%:F"
                            """

                            // Stop + remove ancien container, puis run le nouveau
                            bat """
                                ssh -i "${tmpKey}" ^
                                    -o StrictHostKeyChecking=no ^
                                    ${env.REMOTE_HOST} ^
                                    "sudo docker stop ${env.APP_NAME} 2>/dev/null || true ; sudo docker rm ${env.APP_NAME} 2>/dev/null || true ; sudo docker pull ${env.DOCKER_USER}/${env.APP_NAME}:${env.DOCKER_TAG} && sudo docker run -d --name ${env.APP_NAME} -p 80:80 ${env.DOCKER_USER}/${env.APP_NAME}:${env.DOCKER_TAG}"
                            """

                        } finally {
                            // Supprimer la clé NOUS-MÊMES avant que Jenkins essaie
                            bat "del /f /q \"${tmpKey}\" 2>nul"
                            // Remettre les permissions du fichier Jenkins pour qu'il puisse nettoyer
                            bat "icacls \"%SSH_KEY%\" /grant:r \"%USERNAME%:F\" 2>nul"
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline réussi ! http://192.168.56.103"
            echo "   Image: ${env.DOCKER_USER}/${env.APP_NAME}:${env.DOCKER_TAG}"
        }
        failure {
            echo "❌ Pipeline échoué. Vérifier les logs ci-dessus."
        }
    }
}