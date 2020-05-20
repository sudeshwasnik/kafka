#!/usr/bin/env groovy

/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements. See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License. You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

def config = jobConfig {
    cron = ''
    nodeLabel = 'docker-oraclejdk8'
    testResultSpecs = ['junit': '**/build/test-results/**/TEST-*.xml']
    slackChannel = ''
    timeoutHours = 4
    runMergeCheck = false
    testbreakReporting = false
}

def retryFlagsString(jobConfig) {
    if (jobConfig.isPrJob) " -PmaxTestRetries=1 -PmaxTestRetryFailures=5"
    else ""
}

def downstreamBuildFailureOutput = ""
def publishStep(String configSettings) {
  configFileProvider([configFile(fileId: configSettings, variable: 'GRADLE_NEXUS_SETTINGS')]) {
      // TODO: need to fix this. I disabled signing because it failed, but probably need to enable
      //get it working for these builds since they will be released.
      sh "./gradlewAll -PskipSigning=true --init-script ${GRADLE_NEXUS_SETTINGS} --no-daemon uploadArchives"
  }
}

def job = {
    // https://github.com/confluentinc/common-tools/blob/master/confluent/config/dev/versions.json
    def kafkaMuckrakeVersionMap = [
            "2.3": "5.3.x",
            "2.4": "5.4.x",
            "2.5": "5.5.x",
            "trunk": "master",
            "master": "master"
    ]

    if (!config.isReleaseJob && !config.isPrJob) {
        writeFile file:'.ci/update_version_info.py', text:libraryResource('scripts/update_version_info.py')
        //TODO: fix this, not a good way to install this python library
        runTool("pip install -Uq jprops && python .ci/update_version_info.py --repo-path=${env.WORKSPACE} --repo-name=kafka")
    }

    stage("Check compilation compatibility with Scala 2.12") {
        sh "./gradlew clean assemble spotlessScalaCheck checkstyleMain checkstyleTest spotbugsMain " +
                 "--no-daemon --stacktrace -PxmlSpotBugsReport=true -PscalaVersion=2.12"
        }


    stage("Compile and validate") {
        sh "./gradlew clean assemble spotlessScalaCheck checkstyleMain checkstyleTest spotbugsMain " +
                "--no-daemon --stacktrace -PxmlSpotBugsReport=true"
    }

    if (config.publish && (config.isDevJob || config.isPreviewJob)) {
        stage("Publish to artifactory") {
            if (!config.isReleaseJob && !config.isPrJob) {
                withVaultEnv([["github/confluent_jenkins", "user", "GIT_USER"],
                    ["github/confluent_jenkins", "access_token", "GIT_TOKEN"]]) {
                    withEnv(["GIT_CREDENTIAL=${env.GIT_USER}:${env.GIT_TOKEN}"]) {
                        writeFile file:".ci/setup-credential-store.sh", text:libraryResource('scripts/setup-credential-store.sh')
                        writeFile file:".ci/push_git_tag.py", text:libraryResource('scripts/push_git_tag.py')
                        sh 'bash .ci/setup-credential-store.sh'
                        runTool("python .ci/push_git_tag.py --repo-path=${env.WORKSPACE} --repo-name=kafka")
                    }
                }
            }

            if (config.isDevJob) {
                publishStep('Gradle-Artifactory-Settings')
            } else if (config.isPreviewJob) {
                publishStep('Gradle-Artifactory-Preview-Release-Settings')
            }
        }
    }

    stage("Run Tests and build cp-downstream-builds") {
    	def runTestsStepName = "Step run-tests"
    	def downstreamBuildsStepName = "Step cp-downstream-builds"
        def testTargets = [
           runTestsStepName: {
               stage('Run tests') {
                   echo "Running unit and integration tests"
                   sh "./gradlew unitTest integrationTest " +
                           "--no-daemon --stacktrace --continue -PtestLoggingEvents=started,passed,skipped,failed -PmaxParallelForks=4 -PignoreFailures=true" +
                           retryFlagsString(config)
               }
               stage('Upload results') {
                   // Kafka failed test stdout files
                   archiveArtifacts artifacts: '**/testOutput/*.stdout', allowEmptyArchive: true

                   def summary = junit '**/build/test-results/**/TEST-*.xml'
                   def total = summary.getTotalCount()
                   def failed = summary.getFailCount()
                   def skipped = summary.getSkipCount()
                   summary = "Test results:\n\t"
                   summary = summary + ("Passed: " + (total - failed - skipped))
                   summary = summary + (", Failed: " + failed)
                   summary = summary + (", Skipped: " + skipped)
                   return summary;
               }
        }
        //TODO re-enable when done testing.
        /*,
        downstreamBuildsStepName: {
            echo "Building cp-downstream-builds"
            if (config.isPrJob) {
                try {
                    def muckrakeBranch = kafkaMuckrakeVersionMap[env.CHANGE_TARGET]
                    def forkRepo = "${env.CHANGE_FORK ?: "confluentinc"}/kafka.git"
                    def forkBranch = env.CHANGE_BRANCH
                    echo "Schedule test-cp-downstream-builds with :"
                    echo "Muckrake branch : ${muckrakeBranch}"
                    echo "PR fork repo : ${forkRepo}"
                    echo "PR fork branch : ${forkBranch}"
                    buildResult = build job: 'test-cp-downstream-builds', parameters: [
                            [$class: 'StringParameterValue', name: 'BRANCH', value: muckrakeBranch],
                            [$class: 'StringParameterValue', name: 'TEST_PATH', value: "muckrake/tests/dummy_test.py"],
                            [$class: 'StringParameterValue', name: 'KAFKA_REPO', value: forkRepo],
                            [$class: 'StringParameterValue', name: 'KAFKA_BRANCH', value: forkBranch]],
                            propagate: true, wait: true
                    downstreamBuildFailureOutput = "cp-downstream-builds result: " + buildResult.getResult();
                    return downstreamBuildFailureOutput
                } catch (Exception e) {
                    currentBuild.result = 'UNSTABLE'
                    downstreamBuildFailureOutput = "cp-downstream-builds result: " + e.getMessage()
                    writeFile file: "downstream/cp-downstream-build-failure.txt", text: downstreamBuildFailureOutput
                    archiveArtifacts artifacts: 'downstream/*.txt'

                    return downstreamBuildFailureOutput
                }
            } else {
                return ""
            }
         }*/
        ]

        result = parallel testTargets
        // combine results of the two targets into one result string
        return result.runTestsStepName + "\n" + result.downstreamBuildsStepName
    }
}

runJob config, job
echo downstreamBuildFailureOutput
