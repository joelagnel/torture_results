pipeline {
    agent { label 'x86' }
    environment {
        APPEND_DISPLAY_NAME = 'Default' // Add a custom name to append.
	TRACE_MODE = 'non-tracing' // or 'tracing'
    }

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
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '100', daysToKeepStr: '', numToKeepStr: '30')
    }
    
    stages {
            stage('Checkout') {
                steps {
                    script {
                        echo "Checking out base code"
			sh "if ! test -d .git; then git init; fi"
			sh "git reset HEAD || true"
			sh "git checkout . || true"
			sh "git rebase --abort || true"
			sh "git am --abort || true"
			sh "git clean -f -d || true"
			sh "rm -rf tools/testing/selftests/rcutorture/res/*"
                        sh "git fetch --no-tags --depth=20 git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable-rc.git ${env.BRANCH_NAME}"
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
			    sh '''
       				echo "Checking out OOT code"
			        # Create a unique temporary directory in /tmp/
			        TMP_DIR=$(mktemp -d /tmp/git_patches_XXXXXX)
				rm -rf $TMP_DIR/*
    
			        if git fetch --no-tags --depth=300 https://github.com/joelagnel/linux-kernel.git rcu/$BRANCH_NAME; then
			            STABLE_COMMIT="$(git log --pretty=format:'%H %s' FETCH_HEAD | grep -E 'Linux [1-9][0-9]{0,2}\\.[1-9][0-9]{0,2}.*' | head -n1 | awk '{print $1}')"
			            if [ -n "$STABLE_COMMIT" ]; then
			                git format-patch $STABLE_COMMIT..FETCH_HEAD -o $TMP_DIR
			                git am -3 $TMP_DIR/*
		   		    else echo "No Additional OOT patches being applied to this branch."
			            fi
	       			else echo "No Additional OOT patches being applied to this branch."
			        fi
	   			rm -rf $TMP_DIR
			    '''
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
            steps {
                script {
                    // Print all env variables
                    // sh "printenv"
                    if ("${env.SKIP_TORTURE_TEST}" == "false") {
                        if ("${params.CPURL}" != "none") {
                            currentBuild.displayName = "${env.APPEND_DISPLAY_NAME}-Cherrypick-Test-${env.JOB_NAME}"
                        } else {
                            currentBuild.displayName = "${env.APPEND_DISPLAY_NAME}-${env.JOB_NAME}"
                        }

// This module may not be autoloaded causes qemu to fail, try to load it.
sh "modprobe kvm_intel || true"

/////////// RUNNING KVM.sh NOW ///////////////
if (env.TRACE_MODE == 'non-tracing') {
	// Non-tracing version
	sh "tools/testing/selftests/rcutorture/bin/kvm.sh --cpus 48 --duration 60"
} else {
	// For replay-tracing: Uncomment (remove END_COMMENT) for tracing version of rcutorture
	// The configs and duration can be modified, also change displayName above to differentiate properly.
	sh '''
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
	'''
}
////////////// DONE kvm.sh ///////////////////
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
	    // Delete vmlinux and bzImage due to space
	    sh "find tools/testing/selftests/rcutorture/res/ -name vmlinux | xargs rm"
	    sh "find tools/testing/selftests/rcutorture/res/ -name bzImage | xargs rm"
		
            // TODO: Only extract the res directory corresponding to the currently run test.
            archiveArtifacts artifacts: 'tools/testing/selftests/rcutorture/res/', allowEmptyArchive: 'true'

            script {
                // Define BUILD_LOG at the script's top level
                def BUILD_LOG
    // Use the sh step to check if the file exists and then cat its content
    BUILD_LOG = sh(script: ''' 
        if [ -f tools/testing/selftests/rcutorture/res/*/log ]; then
            cat tools/testing/selftests/rcutorture/res/*/log
        else
            echo ""
        fi
    ''', returnStdout: true).trim()
                def extractLogPart = { log ->
                    def pattern = /.*Test summary:/
                    def matcher = log =~ pattern
                    if (matcher.find()) {
                        return log.substring(matcher.start(0))
                    } else {
                        return log
                    }
                }

                def LOG_PART = extractLogPart(BUILD_LOG)

            // inside script
            emailext (
                body:   '''$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:
                          Check console output at $BUILD_URL to view the results.
                        ''' +
                        """Log summary: ${LOG_PART} """ +
                        '''
                          Console Output:
                          ${BUILD_LOG}
                        ''',
                subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!', 
                to: 'joel@joelfernandes.org'
            )

            } // end script


	    // Clean up the res directory
	    sh "rm -rf tools/testing/selftests/rcutorture/res/*"
        }
    }
}

