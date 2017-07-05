#!groovy

import groovy.time.*

def envList = []
def skipList = []
def startTime = []
def endTime = []
def startTime_b = []
def endTime_b = []
def resultList = []
def resultList_b = []

//List of environments used for build and test. All environments will be run in parallel
envList.add('ORO_PHP=7.1 ORO_APP=application/commerce-crm-ee ORO_TEST_SUITE=behat ORO_DB=pgsql BEHAT_MODE=standalone')
//envList.add('ORO_PHP=7.1 ORO_APP=application/commerce-crm-ee ORO_TEST_SUITE=behat ORO_DB=pgsql')
//envList.add('ORO_PHP=7.1 ORO_APP=application/commerce-crm-ee ORO_TEST_SUITE=behat ORO_DB=mysql')

/* Only keep the 10 most recent builds. */
def projectProperties = [
//    [$class: 'BuildDiscarderProperty',strategy: [$class: 'LogRotator', numToKeepStr: '10']],
    buildDiscarder(logRotator(daysToKeepStr: '15', numToKeepStr: '50')),
]

if (!env.CHANGE_ID) {
    if (env.BRANCH_NAME == null) {
//        projectProperties.add(pipelineTriggers([cron('H/30 * * * *')]))
    }
}

properties(projectProperties)

echo "BRANCH_NAME = ${env.BRANCH_NAME}"
//Use branch name for detect if necessary kill all previous builds OPI-64
if ( ! BRANCH_NAME.startsWith('maintenanace') || ! BRANCH_NAME.startsWith('master')) {
  //kill all job with current name only for PR
  killall_jobs()
}
//Use branch name for detect if necessary run all tests
if (env.BRANCH_NAME.startsWith('PR-')) {
  fullBuild='FULL_BUILD=false'
} else if (env.BRANCH_NAME.startsWith('master')) {
    envList.add('ORO_PHP=7.0 ORO_APP=application/commerce-crm-ee ORO_TEST_SUITE=unit')
    envList.add('ORO_PHP=7.1 ORO_APP=application/commerce-crm-ee ORO_TEST_SUITE=unit ORO_CS=true')
    envList.add('ORO_TEST_SUITE=javascript ORO_CS=true ORO_APP=application/commerce-crm-ee')

    // envList.add('ORO_TEST_SUITE=patch_update ORO_APP=application/commerce-crm-ee')
    // envList.add('ORO_TEST_SUITE=duplicate-queries ORO_APP=application/commerce-crm-ee')

    fullBuild='FULL_BUILD=true'
} else {
  fullBuild='FULL_BUILD=true'
}
echo fullBuild
try {
  timestamps {
    ansiColor('xterm') {
      stage("Checkout") {
        node('master') {
          dir("${HOME}/.ssh/key") {
            stash 'master-stuff'
          }
        }
        if(fullBuild == 'FULL_BUILD=false'){
          node('master') {
            // sh 'docker version && docker info && docker-compose version'
//            checkout_my_onmaster()
            for (int i = 0; i < envList.size() ; i++) {
              int index=i, e = i+1
              withEnv(envList[index].tokenize()) {
                try {
                  checkVal = sh returnStatus: true, script: '''
                    printenv
                    '''
                  echo "checkVal=${checkVal}"
                  if (checkVal == 1) {
                    echo "Skip test ${envList[index]}"
                    skipList.add('skip')
                  }else{
                    echo "Pass test ${envList[index]}"
                    skipList.add('pass')
                  }

                } catch (error) {
                  echo "ERROR: ${error}"
                  throw (error)
                } finally {
//                  archive_all("logs/TestEnv_"+e, env.WORKSPACE)
//                  post_clean_node()
                }
              } //withEnv
            } //for
          } //node
        }else{
          for (int i = 0; i < envList.size() ; i++) {
            skipList.add('pass')
          }
        } //if fullBuild
      } //stage

      withEnv([fullBuild]){
        // def enviroments = [failFast: true]
        def enviroments = [:]
        for (int i = 0; i < envList.size() ; i++) {
          int index=i, e = i+1
          enviroments["TestEnv_${e}"] = {
            stage ("TestEnv_${e} " + envList[index]){
              if (skipList[index] == 'pass') {
                withEnv(["NETWORK=${index}"] + ["TESTENV=TestEnv_${e}"] + envList[index].tokenize()) {
                  if (env.BEHAT_MODE == 'standalone') {
                    node('master') {
                      // JENKINS-6817
                      ws("${HOME}/workspace/dev_${EXECUTOR_NUMBER}") {
                        env.BEHAT_SUIT='OroInstallerBundle'
                        try {
//                          checkout_my()
                          // checkout scm
                          // sh 'printenv'
                          //Run behat installer first time
//                          cache(maxCacheSize: 250, caches: [
//                            [$class: 'ArbitraryFileCache', path: '${HOME}/.cache/composer'],
//                            [$class: 'ArbitraryFileCache', path: '${WORKSPACE}/environment/.composer']
//                            ]) {
                                startTime[index] = new Date()
                                resultList[index] = "PASSED"
                              timeout(time: 60, unit: 'MINUTES') {
                                try {
                                  sh '''
                                    sleep $[ ( $RANDOM % 10 )  + 1 ]s
                                  '''
                                } catch (error) {
                                  resultList[index] = "ERROR"
                                  echo "ERROR: ${error}"
                                  throw (error)
                                } finally {
                                  endTime[index] = new Date()
                                }
                                //Recive suits
                                try {
                                  behatStr = sh (
                                    script: 'echo "OroCustomerBundle OroSaleBundle"',
                                    returnStdout: true
                                  ).trim()

                                } catch (error) {
                                  echo "ERROR: ${error}"
                                  throw (error)
                                }
                              }//timeout
//                            } //cache
                        } catch (error) {
                          echo "ERROR: ${error}"
                          throw (error)
                        } finally {
//                          archive_all("logs/TestEnv_"+e+"_OroInstallerBundle", '/tmp/aufs-mnt_'+env.EXECUTOR_NUMBER)
                          // stash name: 'sources'
//                          dir('/tmp/aufs-tmp_'+env.EXECUTOR_NUMBER)  {
//                            stash name: 'tmp_boot'
//                          }
//                          post_clean_node()
                        }
                      } //workspace
                    }//node
                    behatList=behatStr.tokenize()
                    echo "SUITES=${behatList}"
                    def enviroments_b = [:]
                    for (int j = 0; j < behatList.size() ; j++) {
                      int index_b=j, s = j+1
                      enviroments_b["TestEnv_${e} behat=${behatList[index_b]}"] = {
                        withEnv(["BEHAT_SUIT=${behatList[index_b]}"]) {
                          node('master') {
                            ws("${HOME}/workspace/dev_${EXECUTOR_NUMBER}") {
                              // sh ''' printf %s "$(printenv)" |tr -s '[:space:]' ' ' '''
//                              checkout_my()
                              // deleteDir()
                              // unstash 'sources'
  //                            post_clean_node()
                              sh '''
                                printenv
                              '''
//                              dir('/tmp/aufs-tmp_'+env.EXECUTOR_NUMBER)  {
//                                unstash 'tmp_boot'
//                              }
                                startTime_b[index_b] = new Date()
                                resultList_b[index_b] = "PASSED"
                                try {
                                  timeout(time: 15, unit: 'SECONDS') {
                                    sh '''
                                    echo Run behat $BEHAT_SUIT
                                    sleep $[ ( $RANDOM % 10 )  + 1 ]s
                                    sleep 10

                                  '''
                                  } //timeout
                                } catch (error) {
                                   resultList_b[index_b] = "ERROR"
                                  echo "ERROR: ${error.getMessage()}"
                                  throw (error)
                                }finally {
                                    endTime_b[index_b] = new Date()
//                                  archive_all("logs/TestEnv_"+e+"_"+behatList[index_b], '/tmp/aufs-mnt_'+env.EXECUTOR_NUMBER)
//                                  post_clean_node()
                                }
                            } //workspace
                          } //node
                        } //withEnv
                      } //enviroments_b
                    } //for
                    try {
                      parallel enviroments_b
                    } catch (error) {
                      throw (error)
                    }
                  } else {
                    node('master') {
//                        checkout_my()
//                        cache(maxCacheSize: 250, caches: [
//                            [$class: 'ArbitraryFileCache', path: '${WORKSPACE}/environment/.composer']
//                            ]) {
                      startTime[index] = new Date()
                      resultList[index] = "PASSED"
                      try {
                        timeout(time: 50, unit: 'SECONDS') {
                              sh '''
                                sleep $[ ( $RANDOM % 10 )  + 1 ]s
                              '''
//                          } //cache
                        } //timeout
                      } catch (error) {
                        resultList[index] = "ERROR"
                        echo "ERROR: ${error}"
                        throw (error)
                      } finally {
                        endTime[index] = new Date()
//                        archive_all("logs/TestEnv_"+e, env.WORKSPACE)
                      }
                    } //node
                  } //if behat
                } //withEnv
              } //if pass
            } //stage
            // stage('Report') {
            //
            // }
          } //env
        } //for
        try {
          parallel enviroments
        } catch (error) {
          throw (error)
        }
      }
    }
  }
} catch (error) {
  println (error)
  try {
    emailext body: "${env.BRANCH_NAME} - Build #${env.BUILD_NUMBER} ${env.JOB_NAME} - FAILED!",
    recipientProviders: [[$class: 'FailingTestSuspectsRecipientProvider']],
    subject: "${env.BRANCH_NAME} - Build #${env.BUILD_NUMBER} ${env.JOB_NAME} - FAILED!"
  } catch (error_m) {
    echo "Can't send email"
  }
  throw (error)
} finally {
    node('master') {
        def stats = new StringBuilder("")
        stats.append("Tests name and URL of log, Duration, Result\n")
        stats.append("=========================\n")
        for (int t = 0; t < envList.size() ; t++) {
            if(endTime[t].getClass() == Date && startTime[t].getClass() == Date){
                TimeDuration duration = TimeCategory.minus(endTime[t], startTime[t])
                stats.append("${envList[t]}, ${duration}, ${resultList[t]}\n")
            }else{
                stats.append("${envList[t]}, no time, ${resultList[t]}\n")
            }
        }
        for (int t = 0; t < behatList.size() ; t++) {
            def exec = """ echo "${BUILD_URL}execution/node/\$(grep -l "Run behat ${behatList[t]}" "$JENKINS_HOME/jobs/\$(echo "$JOB_NAME" | sed 's|/|/branches/|g')/builds/$BUILD_ID/"*.log 2>/dev/null | head -n1 | xargs basename | sed 's/\\.log//')/log/" """
            stepURL=sh ( script: exec, returnStdout: true).trim()
            if(endTime_b[t].getClass() == Date && startTime_b[t].getClass() == Date){
                TimeDuration duration = TimeCategory.minus(endTime_b[t], startTime_b[t])
                stats.append("${behatList[t]} ${stepURL}, ${duration}, ${resultList_b[t]}\n")
            }else{
                stats.append("${behatList[t]} ${stepURL}, no time, ${resultList_b[t]}\n")
            }
        }

        stats.append("=========================\n")
        def exec = """
            set +x
            echo "${stats}" | column -s ',' -t
        """
        sh exec
    }
}



def checkout_my() {
  //First clone from google for speed up
  // dir("${HOME}") {
  //   unstash 'master-stuff'
  // }
  try {
    retry(5) {
      // sh '''
      //   gcloud auth activate-service-account jenkins@oro-product-development.iam.gserviceaccount.com --key-file="${HOME}/jenkins@oro-product-development.iam.gserviceaccount.com.json" ||:
      //   [ "$(ls -A .)" ] || time gcloud source repos clone dev . ||:
      // '''
      checkout scm
      // checkout([
      //  $class: 'GitSCM',
      //  branches: scm.branches,
      //  doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
      //  extensions: scm.extensions + [[$class: 'CloneOption', depth: Depth,  noTags: true, reference: '', shallow: Shallow]],
      //  userRemoteConfigs: scm.userRemoteConfigs
      //  ])
    }
  } catch (error) {
    error message:"ERROR: Cannot perform git checkout!, Reason: '${error}'"
  }
}
def checkout_my_onmaster() {
  try {
    retry(5) {
      sh '[ "$(ls -A .)" ] || cp -rf $(pwd)@script/. . ||:'
      checkout scm
    }
  } catch (error) {
    error message:"ERROR: Cannot perform git checkout!, Reason: '${error}'"
  }
}

def archive_all(LogPath='logs', SrcLogs='/tmp/aufs-mnt_0') {
  dir(SrcLogs)  {
    withEnv(["LOGPATH=${LogPath}"]) {
      sh '''
        echo "LOGPATH=${LOGPATH}"
        rm -rf \'logs\'
        mkdir -p "${LOGPATH}"
        if [[ $BEHAT_SUIT ]]; then
          mv -fv "${ORO_APP}/web/uploads/behat/"*.png "${LOGPATH}"/ ||:
          pushd logs
            find . -type f -name *.png -printf '%P\\n' -exec sh -c 'mkdir -p _SCREENSHOTS && ln -s ../"$0" _SCREENSHOTS/"$(printf %s "$0" | sed -e "s/\\//_/g;s/\\._//")"' {} \\;
          popd
          sudo chmod go+r /var/log/php* ||:
          cp -rfv /var/log/php* "${LOGPATH}"/ ||:
          cp -rfv /var/log/nginx "${LOGPATH}"/ ||:
          mv -fv "${HOME}/phantomjs/"*${EXECUTOR_NUMBER}.log "${LOGPATH}"/ ||:
        fi
        mv -fv "${ORO_APP}/app/logs/"* "${LOGPATH}"/ ||:
      '''
    }
    try {
      dir('logs')  {
        archiveArtifacts allowEmptyArchive: true, artifacts: '**', caseSensitive: false
      }
    } catch (error) {
      echo "Archiving artifacts ${error}"
    }
    junit allowEmptyResults: true, testResults: 'logs/**/*.xml'
    sh ''' rm -rf \'logs\' ||: '''
  }
}

def post_clean_node() {
  try {
    retry(5) {
      sh '''
        #Unmount all AUFS
        for dev in $(mount | grep "/tmp/aufs-mnt_${EXECUTOR_NUMBER}" | cut -d' ' -f3); do
          if grep -q "$dev" /proc/mounts; then
            echo "Umounting $dev"
            sudo umount "$dev" || sleep 5
          fi
        done
      '''
    }
  } catch (error) {
    echo "Can't umount AUFS"
  }
}

@NonCPS
def killall_jobs() {
  def jobname = env.JOB_NAME
  def buildnum = env.BUILD_NUMBER.toInteger()

  def job = Jenkins.instance.getItemByFullName(jobname)
  for (build in job.builds) {
    if (!build.isBuilding()) { continue; }
    if (buildnum == build.getNumber().toInteger()) { continue; println "equals" }
    echo "Stop build = ${build}"
    try {
      build.doStop()
    } catch (error) {
      echo "Can't stop ${build}"
    }
  }
}
