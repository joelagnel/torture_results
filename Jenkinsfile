pipeline {
    agent any
    stages {
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

                    currentBuild.displayName = "${env.STABLE_VERSION}"
                    currentBuild.description = "${env.STABLE_VERSION}"

                    // Print all env variables
                    // sh "printenv"

                    if ("${env.SKIP_TORTURE_TEST}" == "false") {
                        echo "Testing with kvm.sh and skip=${env.SKIP_TORTURE_TEST}"
                        sh "tools/testing/selftests/rcutorture/bin/kvm.sh --allcpus --duration 30 --trust-make"
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
            archiveArtifacts artifacts: 'tools/testing/selftests/rcutorture/res/', allowEmptyArchive: 'true'
        }
    }
}
