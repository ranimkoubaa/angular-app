pipeline {
    agent any

    environment {
        DOCKER_TAG = ''
    }

    stages {

        stage('Docker Build') {
            steps {
                script {
                    def rawOutput = bat(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.DOCKER_TAG = rawOutput.tokenize('\n').last().trim()
                    echo "Docker tag: ${env.DOCKER_TAG}"
                }
                bat "docker build -t ranimkoubaa/angular-app:${env.DOCKER_TAG} ."
            }
        }

        stage('DockerHub Push') {
            steps {
                withCredentials([string(credentialsId: 'mydockerhubpassword', variable: 'DockerHubPassword')]) {
                    bat "docker login -u ranimkoubaa -p %DockerHubPassword%"
                }
                bat "docker push ranimkoubaa/angular-app:${env.DOCKER_TAG}"
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'Vagrant_ssh', keyFileVariable: 'SSH_KEY')]) {
                    script {
                        // Copier la clé dans un fichier temporaire qu'on contrôle
                        def tmpKey = "C:\\integ_continue\\tmp_ssh_key_${env.BUILD_NUMBER}.pem"
                        
                        bat """
                            copy "%SSH_KEY%" "${tmpKey}"
                            icacls "${tmpKey}" /inheritance:r
                            icacls "${tmpKey}" /grant:r "%USERNAME%:R"
                        """
                        
                        try {
                            bat """
                                ssh -i "${tmpKey}" ^
                                    -o StrictHostKeyChecking=no ^
                                    jenkins@192.168.56.103 ^
                                    "sudo docker pull ranimkoubaa/angular-app:${env.DOCKER_TAG} && sudo docker stop angular-app || true && sudo docker rm angular-app || true && sudo docker run -d --name angular-app -p 80:80 ranimkoubaa/angular-app:${env.DOCKER_TAG}"
                            """
                        } finally {
                            // On supprime nous-mêmes la clé AVANT que Jenkins essaie
                            bat "del /f /q \"${tmpKey}\""
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline réussi ! Image: ranimkoubaa/angular-app:${env.DOCKER_TAG}"
        }
        failure {
            echo "Pipeline échoué. Vérifier les logs."
        }
    }
}