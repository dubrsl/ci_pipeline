#!groovy

//kill all job with current name
killall_jobs()

def envList = []
def skipList = []

//List of environments used for build and test. All environments will be run in parallel

envList.add('PHP=7.1 APP=application/commerce-crm-ee TEST_SUITE=behat DB=pgsql')
envList.add('PHP=7.1 APP=application/commerce-crm-ee TEST_SUITE=behat DB=mysql PARALLEL_PROCESSES=3')

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
envList.add('PHP=7.1 APP=application/commerce-crm-ee     TEST_SUITE=functional DB=pgsql ORO_INSTALLED=crm-enterprise_1.12 UPGRADE=true')

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

echo "BRANCH_NAME = ${env.BRANCH_NAME}"
if (env.BRANCH_NAME.startsWith('PR-')) {
  fullBuild='FULL_BUILD=false'
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
            //Clean workspace and docker
            // deleteDir()
            sh 'printenv'
            checkout_my()
            for (int i = 0; i < envList.size() ; i++) {
              int index=i
              withEnv(envList[index].tokenize()) {
                checkVal = sh returnStatus: true, script: '''
                  export BUILD_DIR=$(readlink -f ./environment)
                  export TEST_RUNNER_OPTIONS=""
                  export APPLICATION=${APP}
  #                "${BUILD_DIR}/ci/${TEST_SUITE}.sh" check | grep -q "changes were detected" && exit 0 || exit 1
                  '''
                echo "checkVal=${checkVal}"
                if (checkVal == 1) {
                  echo "Skip test ${envList[index]}"
                  skipList.add('skip')
                }else{
                  echo "Pass test ${envList[index]}"
                  skipList.add('pass')
                }
              } //withEnv
            } //for
          } //node
        }else{
          for (int i = 0; i < envList.size() ; i++) {
            skipList.add('pass')
          }
        }//if
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
                    node('behat') {
                      try {
                        checkout_my()
                        // checkout scm
                        // sh 'printenv'
                        //Run behat installer first time
                        cache(maxCacheSize: 250, caches: [
                          [$class: 'ArbitraryFileCache', path: '${HOME}/.cache/composer'],
                          [$class: 'ArbitraryFileCache', path: '${WORKSPACE}/environment/.composer'],
                          [$class: 'ArbitraryFileCache', path: '${HOME}/phantomjs/phantomjs-2.1.1-linux-x86_64/bin']
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
                                  script: 'echo "OroCustomerBundle OroSaleBundle"',
                                  returnStdout: true
                                ).trim()
                                // script: './.jenkins/behat_standalone.sh -l ',
                              } catch (error) {
                                echo "ERROR: ${error}"
                                throw (error)
                              }
                            }
                          }
                          // stash name: 'sources'
                          // dir('/tmp/aufs-tmp_boot')  {
                          //   stash name: 'tmp_boot'
                          // }
                      } catch (error) {
                        echo "ERROR: ${error}"
                        throw (error)
                      } finally {
                        // archive_all("logs/TestEnv_"+e, '/tmp/aufs-mnt_boot')
                        // post_clean_node()
                      }
                    }//node
                    behatList=behatStr.tokenize()
                    echo "SUITES=${behatList}"
                    def enviroments_b = [:]
                    for (int j = 0; j < behatList.size() ; j++) {
                      int index_b=j, s = j+1
                      enviroments_b["TestEnv_${e} behat=${behatList[index_b]}"] = {
                        withEnv(["BEHAT_SUIT=${behatList[index_b]}"]) {
                          node('behat') {
                            // sh ''' printf %s "$(printenv)" |tr -s '[:space:]' ' ' '''
                            checkout_my()
                            // deleteDir()
                            // unstash 'sources'
                            // post_clean_node()
                            // sh '''
                            //   rm -rf /tmp/aufs-tmp_boot && mkdir -p /tmp/aufs-tmp_boot
                            // '''
                            // dir('/tmp/aufs-tmp_boot')  {
                            //   unstash 'tmp_boot'
                            // }
                            timeout(time: 60, unit: 'MINUTES') {
                              cache(maxCacheSize: 250, caches: [
                                [$class: 'ArbitraryFileCache', path: '${HOME}/.cache/composer'],
                                [$class: 'ArbitraryFileCache', path: '${WORKSPACE}/environment/.composer'],
                                [$class: 'ArbitraryFileCache', path: '${HOME}/phantomjs/phantomjs-2.1.1-linux-x86_64/bin']
                                ]) {
                                try {
                                  sh '''
                                    printenv
                                  '''
                                } catch (error) {
                                  echo "ERROR: ${error}"
                                  throw (error)
                                // }finally {
                                  // archive_all("logs/TestEnv_"+e+"_"+behatList[index_b], '/tmp/aufs-mnt_boot')
                                  // post_clean_node()
                                }
                              } //cache
                            } //timeout
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
                      checkout_my()
                      timeout(time: 60, unit: 'MINUTES') {
                        cache(maxCacheSize: 250, caches: [
                          [$class: 'ArbitraryFileCache', path: '${WORKSPACE}/environment/.composer']
                          ]) {
                          try {
                            sh '''
                              printenv
                            '''
                          } catch (error) {
                            echo "ERROR: ${error}"
                            throw (error)
                          // } finally {
                          //   archive_all("logs/TestEnv_"+e, env.WORKSPACE)
                          }
                        } //cache
                      } //timeout
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
  try {
    emailext(body: '${FILE,path="target/failsafe-reports/emailable-report.html"}', mimeType: 'html',
    recipientProviders: [[$class: 'CulpritsRecipientProvider']],
    subject: "${env.JOB_NAME} ${env.BRANCH_NAME} - Build #${env.BUILD_NUMBER} - FAILED!")
  } catch (error_m) {
    echo "Can't send email with error: ${error_m}"
  }
  error error
}


def checkout_my() {
  try {
    //First clone it
    dir("${HOME}") {
      unstash 'master-stuff'
    }
    sh '''
      gcloud auth activate-service-account jenkins@oro-product-development.iam.gserviceaccount.com --key-file="${HOME}/jenkins@oro-product-development.iam.gserviceaccount.com.json" ||:
      [ "$(ls -A .)" ] || time gcloud source repos clone ci-pipeline . ||:
    '''
    checkout scm
    // checkout([
    //  $class: 'GitSCM',
    //  branches: scm.branches,
    //  doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
    //  extensions: scm.extensions + [[$class: 'CloneOption', depth: depth,  noTags: true, reference: '', shallow: shallow]],
    //  userRemoteConfigs: scm.userRemoteConfigs
    //  ])
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
        cp -rfv "${HOME}/phantomjs/"*.log "${LOGPATH}"/ ||:
        cp -rfv environment/ci/artifacts/* "${LOGPATH}"/ ||:
        cp -rfv "${APP}/app/logs/"* "${LOGPATH}"/ ||:
        cp -rfv "${APP}/web/uploads/behat/"*.png "${LOGPATH}"/ ||:
        sudo chmod go+r /var/log/php* ||:
        cp -rfv /var/log/php* "${LOGPATH}"/ ||:
        cp -rfv /var/log/nginx "${LOGPATH}"/ ||:
      '''
    }
    try {
      archiveArtifacts artifacts: 'logs/**'
    } catch (error) {
      echo "Archiving artifacts error"
    }
    junit allowEmptyResults: true, testResults: 'logs/**/*.xml'
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
