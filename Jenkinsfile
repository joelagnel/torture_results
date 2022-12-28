pipeline {
    agent any
    
    // Disable default checkout so that I can have my own checkout stage.
    // A declarative pipeline needs this as the checkout stage is otherwise automatically added.
    options {
        skipDefaultCheckout(true)
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '15', daysToKeepStr: '', numToKeepStr: '15')
    }
    
    stages {
            stage('Checkout') {
                steps {
                    script {
                        echo "Doing scm checkout from Jenkinsfile"
                    }

                    checkout scm

                   /*
                    * A shallow checkout can be done as follows, however it appears to be much slower
                    * than the above normal 'checkout scm'
                    checkout([
                         $class: 'GitSCM',
                          branches: scm.branches,
                          extensions: [[
                           $class: 'CloneOption',
                           shallow: true,
                            depth:   1,
                            timeout: 30
                          ]],
                         userRemoteConfigs: scm.userRemoteConfigs
                    ])
                    */

                    script {
                        echo "Done scm checkout from Jenkinsfile"
                    }
                }
            }
            
        stage('Test') {
            // when {
            // changelog '.*Linux .*rc.*'
            //    expression {  }
            // }
            steps {
                script {
                    env.SKIP_TORTURE_TEST = (sh(returnStatus: true, script: 'git show | grep Linux | grep rc') != 0)
                    env.STABLE_VERSION = (sh(returnStdout: true, script: 'git show | grep Linux | grep rc | sed -e "s/.*Linux //g"'))
                    // env.BUILD_NUMBER = "${env.STABLE_VERSION}"

                    // When doing a cherry-pick replay, prepend this with some name to keep the build less confusion (TODO: make it a parameter)
                    // example: "Real-Cherrypick-Test-${env.STABLE_VERSION}"
                    currentBuild.displayName = "${env.STABLE_VERSION}"
                    currentBuild.description = "${env.STABLE_VERSION}"

                    // Print all env variables
                    // sh "printenv"

                    if ("${env.SKIP_TORTURE_TEST}" == "false") {
                        echo "Testing with kvm.sh"

                        // Add below 2 lines when doing a cherry-pick of a commit (TODO: make the commit a parameter)
                        // echo "Doing cherry-pick first"
                        // sh 'git fetch git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git 96017bf9039763a2e02dcc6adaa18592cd73a39d && git cherry-pick FETCH_HEAD'

                        sh "tools/testing/selftests/rcutorture/bin/kvm.sh --allcpus --duration 30"
                        
                    } else {
                        echo "Skipping build ${env.SKIP_TORTURE_TEST}"
                        currentBuild.result = 'ABORTED'
                    }
                }
            }
        }
    }

    post {
        always {
            // TODO: Only extract the res directory corresponding to the currently run test.
            archiveArtifacts artifacts: 'tools/testing/selftests/rcutorture/res/', allowEmptyArchive: 'true'
        }
    }
}
