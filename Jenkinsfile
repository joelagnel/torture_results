pipeline {
    agent any

    // TODO: Add triggers like this to trigger only at night
    // ref: https://docs.cloudbees.com/docs/admin-resources/latest/pipeline-syntax-reference-guide/declarative-pipeline
    // triggers {
    //    cron('H 4/* 0 0 1-5')
    // }

    parameters {
        string(name: 'CPURL', defaultValue: 'none', description: 'Custom Cherry-Pick Git URL', trim: true)
        string(name: 'CPCOMMIT1', defaultValue: 'none', description: 'Custom Cherry-Pick Commit 1', trim: true)
        string(name: 'CPCOMMIT2', defaultValue: 'none', description: 'Custom Cherry-Pick Commit 2', trim: true)
    }

    
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

                        if ("{params.CPURL}" != "none") {
                            echo "Doing custom cherry-picks"

                            if ("{params.CPCOMMIT1}" != "none") {
                                sh "git fetch ${params.CPURL} ${param.CPCOMMIT1} && git cherry-pick FETCH_HEAD"
                            }

                            if ("{params.CPCOMMIT2}" != "none") {
                                sh "git fetch ${params.CPURL} ${param.CPCOMMIT2} && git cherry-pick FETCH_HEAD"
                            }

                            echo "Done custom cherry-picks"
                        }

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

                    if ("{params.CPURL}" != "none") {
                        currentBuild.displayName = "Cherrypick-Test-${env.STABLE_VERSION}"
                    } else {
                        currentBuild.displayName = "${env.STABLE_VERSION}"
                    }

                    currentBuild.description = "${env.STABLE_VERSION}"

                    // Print all env variables
                    // sh "printenv"

                    if ("${env.SKIP_TORTURE_TEST}" == "false") {
                        echo "Testing with kvm.sh"

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
