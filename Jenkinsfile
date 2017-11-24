/*
    fty-proto - 42ITY core protocols

    Copyright (C) 2014 - 2017 Eaton

    This program is free software; you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation; either version 2 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License along
    with this program; if not, write to the Free Software Foundation, Inc.,
    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
*/

pipeline {
                    agent { label "devel-image" }
    parameters {
        // Use DEFAULT_DEPLOY_BRANCH_PATTERN and DEFAULT_DEPLOY_JOB_NAME if
        // defined in this jenkins setup -- in Jenkins Management Web-GUI
        // see Configure System / Global properties / Environment variables
        // Default (if unset) is empty => no deployment attempt after good test
        // See zproject Jenkinsfile-deploy.example for an example deploy job.
        // TODO: Try to marry MultiBranchPipeline support with pre-set defaults
        // directly in MultiBranchPipeline plugin, or mechanism like Credentials,
        // or a config file uploaded to master for all jobs or this job, see
        // https://jenkins.io/doc/pipeline/examples/#configfile-provider-plugin
        string (
            defaultValue: '${DEFAULT_DEPLOY_BRANCH_PATTERN}',
            description: 'Regular expression of branch names for which a deploy action would be attempted after a successful build and test; leave empty to not deploy. Reasonable value is ^(master|release/.*|feature/*)$',
            name : 'DEPLOY_BRANCH_PATTERN')
        string (
            defaultValue: '${DEFAULT_DEPLOY_JOB_NAME}',
            description: 'Name of your job that handles deployments and should accept arguments: DEPLOY_GIT_URL DEPLOY_GIT_BRANCH DEPLOY_GIT_COMMIT -- and it is up to that job what to do with this knowledge (e.g. git archive + push to packaging); leave empty to not deploy',
            name : 'DEPLOY_JOB_NAME')
        booleanParam (
            defaultValue: true,
            description: 'If the deployment is done, should THIS job wait for it to complete and include its success or failure as the build result (true), or should it schedule the job and exit quickly to free up the executor (false)',
            name: 'DEPLOY_REPORT_RESULT')
        booleanParam (
            defaultValue: true,
            description: 'Attempt build without DRAFT API in this run?',
            name: 'DO_BUILD_WITHOUT_DRAFT_API')
        booleanParam (
            defaultValue: true,
            description: 'Attempt build with DRAFT API in this run?',
            name: 'DO_BUILD_WITH_DRAFT_API')
        booleanParam (
            defaultValue: true,
            description: 'Attempt build with docs in this run?',
            name: 'DO_BUILD_DOCS')
        booleanParam (
            defaultValue: true,
            description: 'Attempt "make check" in this run?',
            name: 'DO_TEST_CHECK')
        booleanParam (
            defaultValue: true,
            description: 'Attempt "make memcheck" in this run?',
            name: 'DO_TEST_MEMCHECK')
        booleanParam (
            defaultValue: true,
            description: 'Attempt "make distcheck" in this run?',
            name: 'DO_TEST_DISTCHECK')
        booleanParam (
            defaultValue: true,
            description: 'Attempt "cppcheck" analysis before this run?',
            name: 'DO_CPPCHECK')
    }
    triggers {
        pollSCM 'H/5 * * * *'
    }
// Note: your Jenkins setup may benefit from similar setup on side of agents:
//        PATH="/usr/lib64/ccache:/usr/lib/ccache:/usr/bin:/bin:${PATH}"
    stages {
        stage ('cppcheck') {
                    when { expression { return ( params.DO_CPPCHECK ) } }
                    steps {
                        sh 'cppcheck --std=c++11 --enable=all --inconclusive --xml --xml-version=2 . 2>cppcheck.xml'
                        archiveArtifacts artifacts: '**/cppcheck.xml'
                    }
        }
        stage ('prepare') {
                    steps {
                        sh './autogen.sh'
                        stash (name: 'prepped', includes: '**/*')
                    }
        }
        stage ('compile') {
            parallel {
                stage ('build with DRAFT') {
                    when { expression { return ( params.DO_BUILD_WITH_DRAFT_API ) } }
                    steps {
                      dir("tmp/build-withDRAFT") {
                        deleteDir()
                        unstash 'prepped'
                        sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; ./configure --enable-drafts=yes'
                        sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make -k -j4 || make'
                        sh 'echo "Are GitIgnores good after make with drafts? (should have no output below)"; git status -s || true'
                        stash (name: 'built-draft', includes: '**/*')
                      }
                    }
                }
                stage ('build without DRAFT') {
                    when { expression { return ( params.DO_BUILD_WITHOUT_DRAFT_API ) } }
                    steps {
                      dir("tmp/build-withoutDRAFT") {
                        deleteDir()
                        unstash 'prepped'
                        sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; ./configure --enable-drafts=no'
                        sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make -k -j4 || make'
                        sh 'echo "Are GitIgnores good after make without drafts? (should have no output below)"; git status -s || true'
                        stash (name: 'built-nondraft', includes: '**/*')
                      }
                    }
                }
                stage ('build with DOCS') {
                    when { expression { return ( params.DO_BUILD_DOCS ) } }
                    steps {
                      dir("tmp/build-DOCS") {
                        deleteDir()
                        unstash 'prepped'
                        sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; ./configure --enable-drafts=yes --with-docs=yes'
                        sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make -k -j4 || make'
                        sh 'echo "Are GitIgnores good after make with docs? (should have no output below)"; git status -s || true'
                      }
                    }
                }
            }
        }
        stage ('check') {
            parallel {
                stage ('check with DRAFT') {
                    when { expression { return ( params.DO_BUILD_WITH_DRAFT_API && params.DO_TEST_CHECK ) } }
                    steps {
                      dir("tmp/test-check-withDRAFT") {
                        deleteDir()
                        unstash 'built-draft'
                        timeout (time: 5, unit: 'MINUTES') {
                            sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make check'
                        }
                        sh 'echo "Are GitIgnores good after make check with drafts? (should have no output below)"; git status -s || true'
                      }
                    }
                }
                stage ('check without DRAFT') {
                    when { expression { return ( params.DO_BUILD_WITHOUT_DRAFT_API && params.DO_TEST_CHECK ) } }
                    steps {
                      dir("tmp/test-check-withoutDRAFT") {
                        deleteDir()
                        unstash 'built-nondraft'
                        timeout (time: 5, unit: 'MINUTES') {
                            sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make check'
                        }
                        sh 'echo "Are GitIgnores good after make check without drafts? (should have no output below)"; git status -s || true'
                      }
                    }
                }
                stage ('memcheck with DRAFT') {
                    when { expression { return ( params.DO_BUILD_WITH_DRAFT_API && params.DO_TEST_MEMCHECK ) } }
                    steps {
                      dir("tmp/test-memcheck-withDRAFT") {
                        deleteDir()
                        unstash 'built-draft'
                        timeout (time: 5, unit: 'MINUTES') {
                            sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make memcheck && exit 0 ; echo "Re-running failed ($?) memcheck with greater verbosity" >&2 ; make VERBOSE=1 memcheck-verbose'
                        }
                        sh 'echo "Are GitIgnores good after make memcheck with drafts? (should have no output below)"; git status -s || true'
                      }
                    }
                }
                stage ('memcheck without DRAFT') {
                    when { expression { return ( params.DO_BUILD_WITHOUT_DRAFT_API && params.DO_TEST_MEMCHECK ) } }
                    steps {
                      dir("tmp/test-memcheck-withoutDRAFT") {
                        deleteDir()
                        unstash 'built-nondraft'
                        timeout (time: 5, unit: 'MINUTES') {
                            sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make memcheck && exit 0 ; echo "Re-running failed ($?) memcheck with greater verbosity" >&2 ; make VERBOSE=1 memcheck-verbose'
                        }
                        sh 'echo "Are GitIgnores good after make memcheck without drafts? (should have no output below)"; git status -s || true'
                      }
                    }
                }
                stage ('distcheck with DRAFT') {
                    when { expression { return ( params.DO_BUILD_WITH_DRAFT_API && params.DO_TEST_DISTCHECK ) } }
                    steps {
                      dir("tmp/test-distcheck-withDRAFT") {
                        deleteDir()
                        unstash 'built-draft'
                        timeout (time: 10, unit: 'MINUTES') {
                            sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make distcheck'
                        }
                        sh 'echo "Are GitIgnores good after make distcheck with drafts? (should have no output below)"; git status -s || true'
                      }
                    }
                }
                stage ('distcheck without DRAFT') {
                    when { expression { return ( params.DO_BUILD_WITHOUT_DRAFT_API && params.DO_TEST_DISTCHECK ) } }
                    steps {
                      dir("tmp/test-distcheck-withoutDRAFT") {
                        deleteDir()
                        unstash 'built-nondraft'
                        timeout (time: 10, unit: 'MINUTES') {
                            sh 'CCACHE_BASEDIR="`pwd`" ; export CCACHE_BASEDIR; make distcheck'
                        }
                        sh 'echo "Are GitIgnores good after make distcheck without drafts? (should have no output below)"; git status -s || true'
                      }
                    }
                }
            }
        }
        stage ('deploy if appropriate') {
            steps {
                script {
                    def myDEPLOY_JOB_NAME = sh(returnStdout: true, script: """echo "${params["DEPLOY_JOB_NAME"]}" """).trim();
                    def myDEPLOY_BRANCH_PATTERN = sh(returnStdout: true, script: """echo "${params["DEPLOY_BRANCH_PATTERN"]}" """).trim();
                    def myDEPLOY_REPORT_RESULT = sh(returnStdout: true, script: """echo "${params["DEPLOY_REPORT_RESULT"]}" """).trim().toBoolean();
                    echo "Original: DEPLOY_JOB_NAME : ${params["DEPLOY_JOB_NAME"]} DEPLOY_BRANCH_PATTERN : ${params["DEPLOY_BRANCH_PATTERN"]} DEPLOY_REPORT_RESULT : ${params["DEPLOY_REPORT_RESULT"]}"
                    echo "Used:     myDEPLOY_JOB_NAME:${myDEPLOY_JOB_NAME} myDEPLOY_BRANCH_PATTERN:${myDEPLOY_BRANCH_PATTERN} myDEPLOY_REPORT_RESULT:${myDEPLOY_REPORT_RESULT}"
                    if ( (myDEPLOY_JOB_NAME != "") && (myDEPLOY_BRANCH_PATTERN != "") ) {
                        if ( env.BRANCH_NAME =~ myDEPLOY_BRANCH_PATTERN ) {
                            def GIT_URL = sh(returnStdout: true, script: """git remote -v | egrep '^origin' | awk '{print \$2}' | head -1""").trim()
                            def GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse --verify HEAD').trim()
                            build job: "${myDEPLOY_JOB_NAME}", parameters: [
                                string(name: 'DEPLOY_GIT_URL', value: "${GIT_URL}"),
                                string(name: 'DEPLOY_GIT_BRANCH', value: env.BRANCH_NAME),
                                string(name: 'DEPLOY_GIT_COMMIT', value: "${GIT_COMMIT}")
                                ], quietPeriod: 0, wait: myDEPLOY_REPORT_RESULT, propagate: myDEPLOY_REPORT_RESULT
                        } else {
                            echo "Not deploying because branch '${env.BRANCH_NAME}' did not match filter '${myDEPLOY_BRANCH_PATTERN}'"
                        }
                    } else {
                        echo "Not deploying because deploy-job parameters are not set"
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                if (currentBuild.getPreviousBuild()?.result != 'SUCCESS') {
                    // Uncomment desired notification

                    //slackSend (color: "#008800", message: "Build ${env.JOB_NAME} is back to normal.")
                    //emailext (to: "qa@example.com", subject: "Build ${env.JOB_NAME} is back to normal.", body: "Build ${env.JOB_NAME} is back to normal.")
                }
            }
        }
        failure {
            // Uncomment desired notification
            // Section must not be empty, you can delete the sleep once you set notification
            sleep 1
            //slackSend (color: "#AA0000", message: "Build ${env.BUILD_NUMBER} of ${env.JOB_NAME} ${currentBuild.result} (<${env.BUILD_URL}|Open>)")
            //emailext (to: "qa@example.com", subject: "Build ${env.JOB_NAME} failed!", body: "Build ${env.BUILD_NUMBER} of ${env.JOB_NAME} ${currentBuild.result}\nSee ${env.BUILD_URL}")
        }
    }
}
