pipeline {
    agent any

    environment {
        // LOCAL TEST: deploy to a folder on this machine, served by python http.server
        // LATER: change this to the sandbox path once SSH access is available
        DEPLOY_DIR = '/tmp/pipeline-test-deploy'
    }

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

        stage('Deploy (Local)') {
            steps {
                echo "Deploying index.html to ${DEPLOY_DIR}..."
                sh '''
                    mkdir -p $DEPLOY_DIR
                    cp index.html $DEPLOY_DIR/index.html
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline succeeded. Serve the deployed file locally with:'
            echo "cd ${DEPLOY_DIR} && python3 -m http.server 8000"
            echo 'Then open http://localhost:8000 in your browser.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above to see which step broke.'
        }
    }
}

/*
LATER — swap the "Deploy (Local)" stage for this once you have sandbox SSH access:

stage('Deploy (Sandbox)') {
    steps {
        sshagent(credentials: ['sandbox-ssh-key']) {
            sh '''
                scp -o StrictHostKeyChecking=no index.html user@SANDBOX_IP:/path/to/webroot/index.html
            '''
        }
    }
}

Everything else in the pipeline (Checkout, Verify File Exists) stays exactly the same.
Only the deploy target changes — that's the whole point of testing locally first.
*/
