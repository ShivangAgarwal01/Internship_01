// ============================================================================
// sandbox-pipeline-test: deploys index.html to the BizKarm sandbox server
//
// Flow:  Checkout -> Verify File Exists -> Deploy -> Verify Deployment -> Health Check
//
// How failure works: every stage runs shell commands. A command that exits with
// anything other than 0 fails its stage, and Jenkins skips every stage after it.
// No AI and no manual review are involved, only exit codes.
//
// Trigger: "Poll SCM" in the job configuration (not in this file) asks GitHub about
// every minute whether main has a new commit. There is no webhook because GitHub
// cannot reach Jenkins running on localhost.
//
// Known shortcuts (acceptable for a static test page, close before real services):
//   - StrictHostKeyChecking=no : the server's identity is not verified
//   - curl -k                  : the HTTPS certificate is not validated
// ============================================================================
pipeline {
    // Run on any available agent. Here that is the Jenkins container itself.
    agent any

    // Stages run top to bottom. The first failure stops the run.
    stages {

        // Pulls the repo and branch this job is configured for (Internship_01, main).
        // Jenkins also fetches the Jenkinsfile itself before this, which is why the
        // console shows two Git fetches. The repeat is harmless.
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }

        // Fail fast: test -f exits 0 if the file exists and 1 if it does not,
        // so nothing is deployed when index.html is missing.
        stage('Verify File Exists') {
            steps {
                echo 'Confirming index.html is present before deploy...'
                sh 'test -f index.html'
            }
        }

        // Copies index.html to the server over SSH. This does not use Webuzo at all.
        // It writes straight into the folder Apache serves for sandbox.bizkarm.in.
        stage('Deploy (Sandbox)') {
            steps {
                // Loads the Jenkins credential into two variables, only inside this block:
                //   SSH_KEY  = path to a temporary file holding the private key
                //   SSH_USER = the username stored with the credential (prodcomtech)
                // Jenkins masks the values in the console log (shown as ****).
                withCredentials([sshUserPrivateKey(credentialsId: 'jenkins-bizkarm-deploy-key',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {
                    // The ''' quotes stop Groovy expanding $SSH_KEY, so the shell does it
                    // and the secret never gets pasted into the script text.
                    sh '''
                        # Copy the key to container-native /tmp with owner-only permissions (600).
                        # The Jenkins workspace is on a Windows-mounted drive that cannot hold real permissions.
                        # (install -m 600 creates the copy with those permissions in one step.)
                        install -m 600 "$SSH_KEY" /tmp/jenkins_deploy_key

                        # Delete the key copy when this shell ends, whether the stage passed or failed.
                        trap 'rm -f /tmp/jenkins_deploy_key' EXIT

                        # scp = copy over SSH.
                        #   -i                            use this private key
                        #   IdentitiesOnly=yes            use only that key, ignore any others
                        #   StrictHostKeyChecking=no      do not ask to confirm the server (nobody can answer a prompt)
                        #   UserKnownHostsFile=/dev/null  do not remember the server's fingerprint
                        # Destination format is user@host:path. The path is the real Apache
                        # document root, plus the pipeline-test subfolder, so the live site is untouched.
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

        // The real content check. A successful scp only means no error was reported.
        // This reads the file back from the server and compares it to the repo copy.
        stage('Verify Deployment') {
            steps {
                echo 'Verifying deployed file matches source...'
                withCredentials([sshUserPrivateKey(credentialsId: 'jenkins-bizkarm-deploy-key',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {
                    sh '''
                        # Same key handling as the Deploy stage.
                        install -m 600 "$SSH_KEY" /tmp/jenkins_deploy_key
                        trap 'rm -f /tmp/jenkins_deploy_key' EXIT

                        # ssh host "command" runs one command on the server. Here it prints the
                        # deployed file, and > saves that output on the Jenkins side.
                        ssh -i /tmp/jenkins_deploy_key \
                            -o IdentitiesOnly=yes \
                            -o StrictHostKeyChecking=no \
                            -o UserKnownHostsFile=/dev/null \
                            "$SSH_USER"@62.72.13.94 \
                            "cat /var/www/html/bizkarm/pipeline-test/index.html" > /tmp/remote_index.html

                        # diff exits 0 if the files are identical and 1 if they differ,
                        # so any mismatch fails the build. Printing nothing means identical.
                        diff index.html /tmp/remote_index.html
                    '''
                }
            }
        }

        // Fetches the live URL. -f makes curl exit non-zero on HTTP errors like 404 or 500.
        // This proves the URL answers. It does not prove the content is fresh, because
        // the Verify Deployment stage does that.
        stage('Health Check') {
            steps {
                // -k skips cert validation: port 442 currently serves an expired cert (known issue).
                sh 'curl -sk -f https://sandbox.bizkarm.in:442/pipeline-test/index.html'
            }
        }

    }

    // post sits beside stages (not inside it). It runs after all stages finish
    // and prints a message based on the result.
    post {
        success {
            echo 'Pipeline succeeded — deployed file verified reachable.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above to see which step broke.'
        }
    }
}