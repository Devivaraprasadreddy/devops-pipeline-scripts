------------------------------------pipeline with playbook-----------------------------------

pipeline {
    agent any

    parameters {
        choice(name: 'BRANCH_NAME', choices: ['main', 'develop'], description: 'Choose the branch to deploy')
    }

    environment {
        REMOTE_HOST = '3.93.190.71'
        REMOTE_USER = 'ubuntu'
        //REMOTE_KEY = 'your-private-key-credential-id'
        REMOTE_PATH = '/home/ubuntu'
        GIT_CREDENTIALS_ID = '369eff06-5c27-4e0b-8116-597a3d04a3af'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    // Checkout the specified branch using Git credentials
                    checkout([$class: 'GitSCM', branches: [[name: "${params.BRANCH_NAME}"]],
                              userRemoteConfigs: [[url: 'https://gitlab.com/naveen51/neogym.git', credentialsId: "${env.GIT_CREDENTIALS_ID}"]]])
                }
            }
        }

        stage('install docker on aws instance') {
            steps {
                script {
                    // install docker on remote aws machine
                        sh """
                        ansible-playbook -i ansible/hosts --private-key /var/lib/jenkins/.ssh/aws -u ubuntu ansible/install_docker.yml						
                        """
                        
                }
            }
        }

        stage('Deploy to Apache') {
            steps {
                script {
                    // Copy files to the remote Apache server
                        sh """
                        rsync -avz  --delete ${WORKSPACE}/ ${env.REMOTE_USER}@${env.REMOTE_HOST}:${env.REMOTE_PATH}
                        """
                    
                }
            }
        }
        
        stage('run docker container') {
            steps {
                script {
                    // run docker container
                        sh """
                        docker pull httpd:latest
                        docker run -d --name container -p 8080:80 -v /home/ubuntu:/usr/local/apache2/htdocs/ httpd:latest					
                        """
                    }
				}
            }
    }
}


