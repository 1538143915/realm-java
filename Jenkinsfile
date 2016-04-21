#!groovy

def isPullRequest() {
    return binding.variables.containsKey('GITHUB_PR_NUMBER')
}

def reportResultToGithub() {
    step([
        $class: 'GitHubPRBuildStatusPublisher',
        statusMsg: [content: "${GITHUB_PR_COND_REF} run ended"],
        unstableAs: 'FAILURE'
    ])
}

def sendMetrics(String metric, String value) {
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: '5b8ad2d9-61a4-43b5-b4df-b8ff6b1f16fa', passwordVariable: 'influx_pass', usernameVariable: 'influx_user']]) {
        sh "curl -i -XPOST 'https://greatscott-pinheads-70.c.influxdb.com:8086/write?db=realm' --data-binary '${metric} value=${value}i' --user '${env.influx_user}:${env.influx_pass}'"
    }
}

def sendABIMetric(String abi) {
    def value = readFile("abi_$abi")
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: '5b8ad2d9-61a4-43b5-b4df-b8ff6b1f16fa', passwordVariable: 'influx_pass', usernameVariable: 'influx_user']]) {
        sh "curl -i -XPOST 'https://greatscott-pinheads-70.c.influxdb.com:8086/write?db=realm' --data-binary 'abi_size,type=${abi} value=${value}i' --user '${env.influx_user}:${env.influx_pass}'"
    }
}

def storeJunitResults(String path) {
    step([
        $class: 'JUnitResultArchiver',
        testResults: path
    ])
}

def collectAarMetrics() {
    dir('realm/realm-library/build/outputs/aar') {
        sh 'unzip realm-android-library-release.aar -d unzipped'
        sh 'find $ANDROID_HOME -name dx | sort -r | head -n 1 > dx'
        sh '$(cat dx) --dex --output=temp.dex unzipped/classes.jar'
        sh 'cat temp.dex | head -c 92 | tail -c 4 | hexdump -e \'1/4 "%d"\' > methods'
        sendMetrics('methods', readFile('methods'))

        sh 'du -k realm-android-library-release.aar | echo -n `cut -f 1` > aar_size'
        sendMetrics('aar_size', readFile('aar_size'))

        sh '''for path in unzipped/jni/* ; do
                abi_size=`du -k $path | cut -f 1`
                abi=`basename $path`
                echo -n $abi_size > abi_$abi
              done
        '''
        sendABIMetric('armeabi')
        sendABIMetric('armeabi-v7a')
        sendABIMetric('arm64-v8a')
        sendABIMetric('mips')
        sendABIMetric('x86')
        sendABIMetric('x86_64')
    }
}

{ ->
    try {
        node('FastLinux') {
           unstash 'java'

           stage 'JVM tests'
           sh 'chmod +x gradlew && ./gradlew assemble check javadoc --stacktrace'
           storeJunitResults('realm/realm-annotations-processor/build/test-results/TEST-*.xml')
           if (env.BRANCH_NAME == 'master') {
               collectAarMetrics()
           }
           /* TODO: add support for running monkey on the example apps
           stash includes: 'examples/*/build/outputs/apk/*debug.apk', name: 'examples' */

           dir('examples') {
              sh 'chmod +x gradlew && ./gradlew check --stacktrace'
              storeJunitResults('unitTestExample/build/test-results/**/TEST-*.xml')
           }

           stage 'static code analysis'
           try {
                dir('realm') {
                   sh 'chmod +x gradlew && ./gradlew findbugs pmd checkstyle --stacktrace'
                }
           } finally {
                publishHTML(target: [allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true, reportDir: 'realm/realm-library/build/findbugs', reportFiles: 'findbugs-output.html', reportName: 'Findbugs issues'])
                publishHTML(target: [allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true, reportDir: 'realm/realm-library/build/reports/pmd', reportFiles: 'pmd.html', reportName: 'PMD Issues'])
           }

           stage 'build instrumented tests'
           dir('realm') {
               sh 'chmod +x gradlew && ./gradlew assembleDebugAndroidTest --stacktrace'
               dir('realm-library/build/outputs/apk') {
                  stash name: 'test-apk', includes: 'realm-android-library-debug-androidTest-unaligned.apk'
               }
           }
        }

        node('android-hub') {
            stage 'run instrumented tests'
            unstash 'test-apk'
            sh 'adb devices'
            sh 'adb devices | grep -v List | grep -v ^$ | awk \'{print $1}\' | parallel \'adb -s {} uninstall io.realm.test; adb -s {} install realm-android-library-debug-androidTest-unaligned.apk; adb -s {} shell am instrument -w -r io.realm.test/android.support.test.runner.AndroidJUnitRunner > test_result_{}.txt; java -jar /opt/log-converter.jar test_result_{}.txt\''
            step([$class: 'JUnitResultArchiver', testResults: 'test_result_*.xml'])

            /* TODO: add support for running monkey on the example apps
            stage 'monkey examples'
            sh 'rm -rf *'
            unstash 'examples' */
        }

        if (env.BRANCH_NAME == 'master') {
            node('FastLinux') {
                stage 'publish to OJO'
                unstash 'java'
                sh 'chmod +x gradlew && ./gradlew assemble ojoUpload'
            }
        }
        currentBuild.rawBuild.setResult(Result.SUCCESS)
    } catch (Exception e) {
        currentBuild.rawBuild.setResult(Result.FAILURE)
    } finally {
        if (isPullRequest()) {
            node {
                reportResultToGithub()
            }
        }
    }
}
