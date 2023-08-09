pipeline {
    agent { label 'x86' }

    // Unconditionally trigger every night (but also we will trigger (in
    // multibranch project settings) by scanning for new changes every 6 hours
    // for branch changes).
    // Note that the unconditional nightly trigger does not care about branch
    // changes, and will build regardless.  ref:
    // https://docs.cloudbees.com/docs/admin-resources/latest/pipeline-syntax-reference-guide/declarative-pipeline
    triggers {
        cron('0 0 * * *')
    }

    parameters {
        string(name: 'CPURL', defaultValue: 'none', description: 'Custom Cherry-Pick Git URL', trim: true)
        string(name: 'CPCOMMIT1', defaultValue: 'none', description: 'Custom Cherry-Pick Commit 1', trim: true)
        string(name: 'CPCOMMIT2', defaultValue: 'none', description: 'Custom Cherry-Pick Commit 2', trim: true)
    }

    
    // Disable default checkout so that I can have my own checkout stage.
    // A declarative pipeline needs this as the checkout stage is otherwise automatically added.
    options {
        skipDefaultCheckout(true)
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '30', daysToKeepStr: '', numToKeepStr: '30')
    }
    
    stages {
            stage('Checkout') {
                steps {
                    // OLD WAY1: Slow
                    // script {
                    //    echo "Doing scm checkout from Jenkinsfile"
                    // }
                    // checkout scm

                   // OLD WAY2: Slow
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

                    // NEW WAY (old school):
                    script {
                        echo "Doing scm checkout from Jenkinsfile"
                        sh "git fetch --no-tags --depth=5000 git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable-rc.git ${env.BRANCH_NAME}"
                        sh "git checkout FETCH_HEAD"
                    }
                    
                    script {
                        env.SKIP_TORTURE_TEST = (sh(returnStatus: true, script: 'git show | egrep "Linux [0-9]+\\.[0-9]+\\.[0-9]+"') != 0)

                        if ("${env.SKIP_TORTURE_TEST}" == "true") {
                            error("Aborting due to SKIP_TORTURE_TEST being set to true")
                        } else {
                            // The head commands are to make JOB_NAME extraction for grafted commits work where the entire diff is in the changelog.
                            // Since the entire diff is in the change log, this causes several hits to lines with the Linux pattern.
                            env.JOB_NAME = (sh(returnStdout: true, script: 'git show | head -n 100 | egrep "Linux [0-9]+\\.[0-9]+\\.[0-9]+" | head -n1 | sed -e "s/.*Linux //g"'))
                            currentBuild.description = "Stable kernel testing"
                        }

                        // env.BUILD_NUMBER = "${env.JOB_NAME}"
                    }

                    /* Merge my staging branch for OOT-stable patches, if it exists. */
                    script {
                        sh """
                            if git fetch --no-tags --depth=5000 https://github.com/joelagnel/linux-kernel.git rcu/$BRANCH_NAME; then
                                git merge FETCH_HEAD
                            fi
                        """
                    }

                    script {

                        if ("${params.CPURL}" != "none") {
                            echo "Doing custom cherry-picks"

                            if ("${params.CPCOMMIT1}" != "none") {
                                sh "git fetch ${params.CPURL} ${params.CPCOMMIT1} && git cherry-pick FETCH_HEAD"
                            }

                            if ("${params.CPCOMMIT2}" != "none") {
                                sh "git fetch ${params.CPURL} ${params.CPCOMMIT2} && git cherry-pick FETCH_HEAD"
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

                    // Print all env variables
                    // sh "printenv"

                    if ("${env.SKIP_TORTURE_TEST}" == "false") {
                        if ("${params.CPURL}" != "none") {
                            currentBuild.displayName = "Cherrypick-Test-${env.JOB_NAME}"
                        } else {
                            currentBuild.displayName = "${env.JOB_NAME}"
                        }

                        echo "Testing with kvm.sh"

// This module may not be autoloaded causes qemu to fail, try to load it.
sh "modprobe kvm_intel || true"

// Non-tracing version
sh "tools/testing/selftests/rcutorture/bin/kvm.sh --cpus 48 --duration 60"

// For replay-tracing: Uncomment (remove END_COMMENT) for tracing version of rcutorture
// The configs and duration can be modified, also change displayName above to differentiate properly.
sh '''
: <<'END_COMMENT'

# Define the kconfig array
kconfigs=(
    "CONFIG_RCU_TRACE=y"
    "CONFIG_PROVE_LOCKING=y"
    "CONFIG_DEBUG_LOCK_ALLOC=y"
    "CONFIG_DEBUG_INFO_DWARF5=y"
    "CONFIG_RANDOMIZE_BASE=n"
    "CONFIG_LOCKUP_DETECTOR=y"
    "CONFIG_SOFTLOCKUP_DETECTOR=y"
    "CONFIG_HARDLOCKUP_DETECTOR=y"
    "CONFIG_DETECT_HUNG_TASK=y"
    "CONFIG_DEFAULT_HUNG_TASK_TIMEOUT=60"
)

# Define the bootargs array (excluding trace_event)
bootargs=(
    "ftrace_dump_on_oops"
    "panic_on_warn=1"
    "sysctl.kernel.panic_on_rcu_stall=1"
    "sysctl.kernel.max_rcu_stall_to_panic=1"
    "trace_buf_size=10K"
    "traceoff_on_warning=1"
    "panic_print=0x1f"      # To dump held locks, mem and other info.
)

# Define the trace events array passed to bootargs.
trace_events=(
    "sched:sched_switch"
    "sched:sched_waking"
    "rcu:rcu_callback"
    "rcu:rcu_fqs"
    "rcu:rcu_quiescent_state_report"
    "rcu:rcu_grace_period"
)

# Call kvm.sh with the arrays
tools/testing/selftests/rcutorture/bin/kvm.sh \
    --cpus 48 \
    --duration 120 \
    --configs "2*TREE04" \
    --kconfig "$(IFS=" "; echo "${kconfigs[*]}")" \
    --bootargs "trace_event=$(IFS=,; echo "${trace_events[*]}") $(IFS=" "; echo "${bootargs[*]}")"

END_COMMENT
'''                        
                    } else {
                        echo "Skipping build ${env.SKIP_TORTURE_TEST}"
                        currentBuild.displayName = "Skipped as ToT is not a stable commit (Linux...)"
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
            
            emailext (
                body: '''$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:
                          Check console output at $BUILD_URL to view the results.

                          Console Output:
                          ${BUILD_LOG}

                          ''',
                subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!', 
                to: 'joel@joelfernandes.org'
            )
        }
    }
}

