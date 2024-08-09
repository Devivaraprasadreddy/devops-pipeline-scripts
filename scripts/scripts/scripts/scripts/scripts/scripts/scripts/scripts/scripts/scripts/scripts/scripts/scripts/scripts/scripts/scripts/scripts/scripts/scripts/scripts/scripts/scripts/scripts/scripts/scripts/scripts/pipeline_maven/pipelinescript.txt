pipeline {
    agent any
    
    tools {
        maven '3.6.3' // Name of the Maven installation in Jenkins
    }

    parameters {
        choice(name: 'BRANCH_NAME', choices: ['main'], description: 'Choose the branch to deploy')
    }

    environment {
        REMOTE_HOST = '192.168.29.130'
        REMOTE_USER = 'root'
        REMOTE_KEY = '/var/lib/jenkins/.ssh/maven'
        REMOTE_PATH = '/root/mnt/data'
        GIT_CREDENTIALS_ID = '0ce9cbdc-1c6e-4cb1-866b-bbec3ffb6db3'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    // Checkout the specified branch using Git credentials
                    checkout([$class: 'GitSCM', branches: [[name: "${params.BRANCH_NAME}"]],
                              userRemoteConfigs: [[url: 'https://gitlab.com/dsp9391/maven_project.git', credentialsId: "${env.GIT_CREDENTIALS_ID}"]]])
                }
            }
        }
        
        stage('Build') {
            steps {
                // Build the project using Maven
                sh 'mvn clean package'
            }
        }

        stage('Deploy to tomcat') {
            steps {
                script {
                    // Copy files to the remote Apache server
                        sh """
                        cd target
                        rsync -avz -e 'ssh -i ${REMOTE_KEY}'  my-webapp.war  ${env.REMOTE_USER}@${env.REMOTE_HOST}:${env.REMOTE_PATH}
                        ssh -i ${REMOTE_KEY} root@192.168.29.130 'docker stop tomcat'
                         ssh -i ${REMOTE_KEY} root@192.168.29.130 'docker rm tomcat'
                        ssh -i ${REMOTE_KEY} root@192.168.29.130 'docker run -d --name tomcat -p 8080:8080 -v /root/mnt/data:/usr/local/tomcat/webapps tomcat'

                        """
                    
                }
            }
        }
    }

    post {
        always {
            echo 'Deployment completed.'
        }
    }
}
