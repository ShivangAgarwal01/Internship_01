pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }

        stage('Verify File Exists') {
            steps {
                echo 'Confirming index.html is present before deploy...'
                sh 'test -f index.html'
            }
        }

        stage('Deploy (Sandbox)') {
            steps {
                sshagent(credentials: ['jenkins-bizkarm-deploy-key']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no index.html \
                            prodcomtech@62.72.13.94:/var/www/html/bizkarm/pipeline-test/index.html
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Verifying deployed file matches source...'
                sshagent(credentials: ['jenkins-bizkarm-deploy-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no prodcomtech@62.72.13.94 \
                            "cat /var/www/html/bizkarm/pipeline-test/index.html" > /tmp/remote_index.html
                        diff index.html /tmp/remote_index.html
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh 'curl -sk https://sandbox.bizkarm.in:442/pipeline-test/index.html -H "Host: sandbox.bizkarm.in" -f'
            }
        }

    }

    post {
        success {
            echo 'Pipeline succeeded — deployed file verified reachable.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above to see which step broke.'
        }
    }
}
