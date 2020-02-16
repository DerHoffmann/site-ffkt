node {
    def GLUON_BRANCH = "v2019.1.1"
    def GLUON_RELEASE = ""
    def PROCS = sh(script: 'nproc', returnStdout: true).toInteger() + 2
    def gluon = {}

    stage("pull") {
        gluon = checkout([$class: 'GitSCM', branches: [[name: "*/${GLUON_BRANCH}.x"]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'gluon'],[$class: 'PathRestriction', excludedRegions: '''docs/\\.*
.*\\.md''', includedRegions: '']], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/freifunk-gluon/gluon.git']]])
        checkout([$class: 'GitSCM', branches: [[name: "*/${GLUON_BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'PathRestriction', excludedRegions: '''docs/\\.*
.*\\.md''', includedRegions: '']], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/DerHoffmann/site-ffkt.git']]])
        dir("gluon") {
            sh 'ln -nfs ../ site'
        }
    }

    def build_env
    stage("build env") {
        build_env = docker.build("gluon-build", "docker")
    }

    build_env.inside {
        dir('gluon') {
            stage('gluon update') {
                if ("$gluon.GIT_PREVIOUS_SUCCESSFUL_COMMIT" != "$gluon.GIT_COMMIT") {
                    sh 'make update'
                }
                GLUON_RELEASE = sh (script: 'make show-release', returnStdout: true).trim()
                currentBuild.displayName = "#${BUILD_NUMBER} - ${GLUON_RELEASE}"
            }
            def targets = sh(script: 'make list-targets', returnStdout: true).split("\n")
            for (int i = 0; i < targets.length; i++) {
                stage("build ${targets[i]}") {
                    withEnv(["GLUON_TARGET=${targets[i]}"]) {
                        sh "make -j ${PROCS} || make V=s"
                    }
                }
            }

            stage('sign') {
                sh "make manifest GLUON_PRIORITY=14 GLUON_BRANCH=stable"
                sh "make manifest GLUON_PRIORITY=7  GLUON_BRANCH=beta"
                sh "make manifest GLUON_PRIORITY=3  GLUON_BRANCH=experimental"
                archiveArtifacts artifacts: 'output/images/sysupgrade/*.manifest', fingerprint: true

                withCredentials([file(credentialsId: 'a405a0c0-7471-4966-be77-2d62b6617dfa', variable: 'ECDSAKEY')]) {
                    sh "bash contrib/sign.sh $ECDSAKEY output/images/sysupgrade/stable.manifest"
                    sh "bash contrib/sign.sh $ECDSAKEY output/images/sysupgrade/beta.manifest"
                    sh "bash contrib/sign.sh $ECDSAKEY output/images/sysupgrade/experimental.manifest"
                }
            }

        }
    }
    stage('upload') {
        sh "rsync -vrlue ssh gluon/output/images/. root@srv1.freifunk-vec.de::firmware/${GLUON_RELEASE}"
        sh "rsync -vrlce ssh gluon/output/packages/. root@rv1.freifunk-vec.de::packages"
    }
}
