#!groovy

def envList = []
def skipList = []

//List of environments used for build and test. All environments will be run in parallel
envList.add('PHP=7.1 APP=application/commerce-crm-ee TEST_SUITE=behat DB=pgsql')
// envList.add('PHP=7.1 APP=application/commerce-crm-ee TEST_SUITE=behat DB=mysql PARALLEL_PROCESSES=3')

envList.add('PHP=7.0 APP=application/commerce-crm-ee TEST_SUITE=unit')
envList.add('PHP=7.1 APP=application/commerce-crm-ee TEST_SUITE=unit CS=true')
envList.add('TEST_SUITE=javascript CS=true APP=application/commerce-crm-ee')

// envList.add('TEST_SUITE=documentation APP=documentation/crm')
// envList.add('TEST_SUITE=documentation APP=documentation/commerce')

// envList.add('TEST_SUITE=patch_update APP=application/commerce-crm-ee')
// envList.add('TEST_SUITE=duplicate-queries APP=application/commerce-crm-ee')

// envList.add('PHP=7.0 APP=application/crm-enterprise      TEST_SUITE=functional DB=pgsql ORO_INSTALLED=crm-enterprise_1.12 UPGRADE=true')
// envList.add('PHP=7.0 APP=application/commerce-enterprise TEST_SUITE=functional DB=mysql ORO_INSTALLED=commerce_1.0.0')

// envList.add('PHP=7.1 APP=application/platform            TEST_SUITE=functional DB=pgsql')
// envList.add('PHP=7.1 APP=application/crm                 TEST_SUITE=functional DB=mysql')

// envList.add('PHP=7.1 APP=application/commerce            TEST_SUITE=functional DB=pgsql ORO_INSTALLED=commerce_1.0.0')
// envList.add('PHP=7.1 APP=application/commerce-crm        TEST_SUITE=functional DB=pgsql ORO_ENABLE_PRICE_SHARDING=true')

// envList.add('PHP=7.1 APP=application/crm-enterprise      TEST_SUITE=functional DB=pgsql ORO_INSTALLED=crm-enterprise_1.12  ORO_SEARCH_ENGINE=elastic_search ELASTIC_SEARCH=2.3 TEST_RUNNER_OPTIONS=--group=search,dist UPGRADE=true')
// envList.add('PHP=7.1 APP=application/crm-enterprise      TEST_SUITE=functional DB=pgsql ORO_INSTALLED=crm-enterprise_2.0.0 ORO_SEARCH_ENGINE=elastic_search')
// envList.add('PHP=7.1 APP=application/commerce-enterprise TEST_SUITE=functional DB=pgsql ORO_INSTALLED=commerce_1.1.0       ORO_SEARCH_ENGINE=elastic_search')

// envList.add('PHP=7.1 APP=application/commerce-crm-ee     TEST_SUITE=functional DB=mysql ORO_INSTALLED=commerce_1.1.0')
// envList.add('PHP=7.1 APP=application/commerce-crm-ee     TEST_SUITE=functional DB=pgsql ORO_INSTALLED=crm-enterprise_1.12 UPGRADE=true')

/* Only keep the 10 most recent builds. */
def projectProperties = [
    [$class: 'BuildDiscarderProperty',strategy: [$class: 'LogRotator', numToKeepStr: '10']],
]

if (!env.CHANGE_ID) {
    if (env.BRANCH_NAME == null) {
        projectProperties.add(pipelineTriggers([cron('H/30 * * * *')]))
    }
}

properties(projectProperties)

//Use branch name for detect if necessary run all tests
echo "BRANCH_NAME = ${env.BRANCH_NAME}"
if (env.BRANCH_NAME.startsWith('PR-')) {
  fullBuild='FULL_BUILD=false'
  //kill all job with current name only for PR
  killall_jobs()
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
          node('behat') {
            ws("${HOME}/workspace/dev") {
              // sh 'docker version && docker info && docker-compose version'
              checkout_my()
              for (int i = 0; i < envList.size() ; i++) {
                int index=i, e = i+1
                withEnv(envList[index].tokenize()) {
                  try {
                    checkVal = sh returnStatus: true, script: '''
                      exit 0
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
                    // archive_all("logs/TestEnv_"+e, env.WORKSPACE)
                    // post_clean_node()
                  }
                } //withEnv
              } //for
            } // ws
          } //node
        }else{
          for (int i = 0; i < envList.size() ; i++) {
            skipList.add('pass')
          }
        } //if fullBuild
      } //stage

      withEnv(["PARALLEL_PROCESSES=8", fullBuild]){
        // def enviroments = [failFast: true]
        def enviroments = [:]
        for (int i = 0; i < envList.size() ; i++) {
          int index=i, e = i+1
          enviroments["TestEnv_${e}"] = {
            stage ("TestEnv_${e} " + envList[index]){
              if (skipList[index] == 'pass') {
                withEnv(["NETWORK=${index}"] + ["TESTENV=TestEnv_${e}"] + envList[index].tokenize()) {
                  if (env.TEST_SUITE == 'behat') {
                    node('master') {
                      ws("${HOME}/workspace/dev") {
                        try {
                          checkout_my()
                          // checkout scm
                          // sh 'printenv'
                          //Run behat installer first time
                          cache(maxCacheSize: 250, caches: [
                            [$class: 'ArbitraryFileCache', path: '${HOME}/.cache/composer'],
                            [$class: 'ArbitraryFileCache', path: '${WORKSPACE}/environment/.composer']
                            ]) {
                              timeout(time: 60, unit: 'MINUTES') {
                                try {
                                  sh '''
                                  printenv
                                  '''
                                } catch (error) {
                                  echo "ERROR: ${error}"
                                  throw (error)
                                }
                                //Recive suits
                                try {
                                  behatStr = sh (
                                    script: 'echo suit1 suit2',
                                    returnStdout: true
                                  ).trim()
                                  // script: 'echo "OroCustomerBundle OroSaleBundle"',
                                } catch (error) {
                                  echo "ERROR: ${error}"
                                  throw (error)
                                }
                              }
                            }
                        } catch (error) {
                          echo "ERROR: ${error}"
                          throw (error)
                        } finally {
                          // archive_all("logs/TestEnv_"+e, '/tmp/aufs-mnt_boot')
                          // post_clean_node()
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
                            ws("${HOME}/workspace/dev") {
                              // sh ''' printf %s "$(printenv)" |tr -s '[:space:]' ' ' '''
                              checkout_my()
                              // deleteDir()
                              // unstash 'sources'
                              timeout(time: 60, unit: 'MINUTES') {
                                cache(maxCacheSize: 250, caches: [
                                  [$class: 'ArbitraryFileCache', path: '${HOME}/.cache/composer'],
                                  [$class: 'ArbitraryFileCache', path: '${WORKSPACE}/environment/.composer']
                                  ]) {
                                  try {
                                    sh '''
                                      printenv
                                      echo ===== run ${BEHAT_SUIT} =====
                                    '''
                                  } catch (error) {
                                    echo "ERROR: ${error}"
                                    throw (error)
                                  }finally {
                                    // archive_all("logs/TestEnv_"+e+"_"+behatList[index_b], '/tmp/aufs-mnt_boot')
                                    // post_clean_node()
                                  }
                                } //cache
                              } //timeout
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
                    node('docker') {
                      try {
                        checkout_my()
                        timeout(time: 60, unit: 'MINUTES') {
                          cache(maxCacheSize: 250, caches: [
                            [$class: 'ArbitraryFileCache', path: '${WORKSPACE}/environment/.composer']
                            ]) {
                              sh '''
                                printenv
                                echo ===== ${TEST_SUITE} ${APP} =====
                              '''
                          } //cache
                        } //timeout
                      } catch (error) {
                        echo "ERROR: ${error}"
                        throw (error)
                      } finally {
                        // archive_all("logs/TestEnv_"+e, env.WORKSPACE)
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
        }
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
    recipientProviders: [[$class: 'UpstreamComitterRecipientProvider']],
    subject: "${env.BRANCH_NAME} - Build #${env.BUILD_NUMBER} ${env.JOB_NAME} - FAILED!"
  } catch (error_m) {
    echo "Can't send email"
  }
  throw (error)
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

def archive_all(LogPath='logs', SrcLogs='/tmp/aufs-mnt_boot') {
  dir(SrcLogs)  {
    withEnv(["LOGPATH=${LogPath}"]) {
      sh '''
        echo "LOGPATH=${LOGPATH}"
        rm -rf \'logs\'
        mkdir -p "${LOGPATH}"
        #docker-compose -f ${COMPOSE_FILE} -p ${PROJECT_NAME} logs --no-color --timestamps > ${LOGPATH}/docker.log ||:
        mv -fv "${HOME}/phantomjs/"*.log "${LOGPATH}"/ ||:
        mv -fv environment/ci/artifacts/* "${LOGPATH}"/ ||:
        mv -fv "${APP}/app/logs/"* "${LOGPATH}"/ ||:
        mv -fv "${APP}/web/uploads/behat/"*.png "${LOGPATH}"/ ||:
        sudo chmod go+r /var/log/php* ||:
        cp -rfv /var/log/php* "${LOGPATH}"/ ||:
        cp -rfv /var/log/nginx "${LOGPATH}"/ ||:
      '''
    }
    try {
      archiveArtifacts artifacts: 'logs/**'
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
        for dev in $(mount | grep '/tmp/aufs' | cut -d' ' -f3); do
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
    echo "Kill task = ${build}"
    build.doStop();
  }
}
