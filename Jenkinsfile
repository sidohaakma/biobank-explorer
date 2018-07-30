pipeline {
    agent {
        kubernetes {
            label 'node-carbon'
        }
    }
    environment {
        ORG = 'molgenis'
    }
    stages {
        stage('Prepare') {
            steps {
                script {
                    env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                }
            }
        }
        stage('Install and test: [ pull request ]') {
            when {
                changeRequest()
            }
            steps {
                container('node') {
                    sh "yarn install"
                    sh "yarn test"
                }
            }
            post {
                always {
                    sh "curl -s https://codecov.io/bash | bash -s - -c -F unit -K"
                }
            }
        }
        stage('Install, test and build: [ master ]') {
            when {
                branch 'master'
            }
            steps {
                milestone 1
                container('node') {
                    sh "yarn install"
                    sh "yarn test"
                }
            }
            post {
                always {
                    sh "curl -s https://codecov.io/bash | bash -s - -c -F unit -K"
                }
            }
        }
        stage('Release: [ master ]') {
            when {
                branch 'master'
            }
            environment {
                REGISTRY = "registry.molgenis.org"
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        env.RELEASE_SCOPE = input(
                                message: 'Do you want to release?',
                                ok: 'Release',
                                parameters: [
                                        choice(choices: 'patch\nminor\nmajor', description: '', name: 'RELEASE_SCOPE')
                                ]
                        )
                    }
                }
                milestone 2
                container('node') {
                    sh """
                       export APP_NAME=$(node -pe \"require('./package.json').name\")

                       git config --global user.email git@molgenis.org
                       git config --global user.name molgenis
                       git remote set-url origin https://${env.GITHUB_TOKEN}@github.com/${ORG}/${APP_NAME}.git

                       git checkout -f master

                       npm version ${env.RELEASE_SCOPE}

                       git push --tags origin master

                       export APP_VERSION=$(node -pe \"require('./package.json').version\")
                       curl -v --user '${env.NEXUS_USERNAME}:${env.NEXUS_PASSWORD}' --upload-file dist/${APP_NAME}.zip  https://${REGISTRY}/repository/appstore/${APP_NAME}/${APP_VERSION}/${APP_NAME}.zip
                    """
                }
            }
        }
    }
    post {
        // [ slackSend ]; has to be configured on the host, it is the "Slack Notification Plugin" that has to be installed
        success {
            notifySuccess()
        }
        failure {
            notifyFailed()
        }
    }
}

def notifySuccess() {
    slackSend(channel: '#releases', color: '#00FF00', message: "RPM-build is successfully deployed on https://registry.npmjs.org: Job - <${env.BUILD_URL}|${env.JOB_NAME}> | #${env.BUILD_NUMBER}")
}

def notifyFailed() {
    slackSend(channel: '#releases', color: '#FF0000', message: "RPM-build has failed: Job - <${env.BUILD_URL}|${env.JOB_NAME}> | #${env.BUILD_NUMBER}")
}
