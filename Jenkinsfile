pipeline {

    options {
        timestamps()
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(
                logRotator(
                        artifactDaysToKeepStr: '10',
                        artifactNumToKeepStr: '5',
                        daysToKeepStr: '10',
                        numToKeepStr: '5'
                )
        )
        ansiColor('xterm')
        parallelsAlwaysFailFast()
    }

    agent {
        docker {
            image 'docker.seal-software.net/build-agent'
            args dockerArgs()
            label 'DO-677'
        }
    }

    environment {
        VERSION = buildVersion()
        FRAMEWORK_VERSION = "2.0.11-${env.BUILD_NUMBER}"
    }

    stages {

        stage('setup') {
            steps {
                sh 'mvn -V -B versions:set -DnewVersion=$VERSION -DgenerateBackupPoms=false'
            }
        }

        stage('build') {
            steps {
                sh "mvn clean clover:setup test clover:aggregate clover:clover -U -B -V -Dmaven.source.skip=true -Pclover.report"
            }
            post {
                always {
                    // Show test results in jenkins ui
                    junit testResults: "**/target/reports/**/*.xml", allowEmptyResults: true
                }
                success {
                    stash name: 'unit-tests-for-sonar', includes: '**/target/reports/**/*.xml,**/target/site/clover/clover.xml'

                    sh "mvn clean install -DskipTests=true -B -V -e"
                }
            }
        }

        stage('integration-test') {
            steps {
                sh "mvn verify -B -V -Dmaven.source.skip=true -DskipITs=false -DskipUTs=true"
            }
        }

        stage('deploy framework') {
            when {
                expression {
                    return !isPrBuild()
                }
            }
            steps {
                deployFramework()
            }
        }

        stage('sonar') {
            when {
                branch 'master'
            }
            steps {
                unstash name: 'unit-tests-for-sonar'
                sonarMvn2()
            }
        }
    }

    post {
        cleanup {
            cleanWs()
            cleanDocker()
        }
    }
}

def deployFramework() {
    def mProfile = getBuildProfile()
    def mVersion = isMaster() ? FRAMEWORK_VERSION : VERSION
    println("Deploying framework with version - $mVersion")
    sh("mvn -B versions:set -DnewVersion=$mVersion -DgenerateBackupPoms=false -f pom.xml")
    sh("mvn -B -e -P$mProfile,skipTests -Dmaven.test.skip=true deploy")
}

def getBuildProfile() {
    return isMasterOrRelease() ? "release" : "builds"
}

def isMasterOrRelease() {
    return isMaster() || env.BRANCH_NAME ==~ /^releases\/.*/ || env.BRANCH_NAME ==~ /^....Q.$/
}

def isMaster() {
    return env.BRANCH_NAME == 'master'
}
