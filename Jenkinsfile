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
                withCredentials([sshUserPrivateKey(credentialsId: 'jenkins-bizkarm-deploy-key',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {
                    sh '''
                        # Copy the key to container-native /tmp with owner-only permissions (600).
                        # The Jenkins workspace is on a Windows-mounted drive that cannot hold real permissions.
                        install -m 600 "$SSH_KEY" /tmp/jenkins_deploy_key
                        trap 'rm -f /tmp/jenkins_deploy_key' EXIT

                        scp -i /tmp/jenkins_deploy_key \
                            -o IdentitiesOnly=yes \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            index.html \
                            "$SSH_USER"@62.72.13.94:/var/www/html/bizkarm/pipeline-test/index.html
                    '''
                }
            }
        }

        stage('Ver Deployment') {
            steps {
                echo 'Verifying deployed file matches source...'
                withCredentials([sshUserPrivateKey(credentialsId: 'jenkins-bizkarm-deploy-key',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {
                    sh '''
                        install -m 600 "$SSH_KEY" /tmp/jenkins_deploy_key
                        trap 'rm -f /tmp/jenkins_deploy_key' EXIT

                        ssh -i /tmp/jenkins_deploy_key \
                            -o IdentitiesOnly=yes \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            "$SSH_USER"@62.72.13.94 \
                            "cat /var/www/html/bizkarm/pipeline-test/index.html" > /tmp/remote_index.html

                        diff index.html /tmp/remote_index.html
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                // -k skips cert validation: port 442 currently serves an expired cert (known issue).
                sh 'curl -sk -f https://sandbox.bizkarm.in:442/pipeline-test/index.html'
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