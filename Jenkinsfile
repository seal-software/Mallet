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

    parameters {
        booleanParam(name: "DoDeploy", defaultValue: "false", description: "Deploy docker image")
    }

    agent {
        docker {
            image 'docker.seal-software.net/build-agent'
            args dockerArgs()
            label 'medium'
        }
    }

    environment {
        VERSION = buildVersion()
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

        stage('sonar') {
            when {
                branch 'master'
            }
            steps {
                unstash name: 'unit-tests-for-sonar'
                sonarMvn2()
            }
        }

        stage('deploy to docker registry') {
            when {
                expression { return params.DoDeploy }
            }
            environment {
                VERSION = "${buildVersion()}"
            }
            steps {
                sh "./docker-build.sh ${VERSION} true"
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
