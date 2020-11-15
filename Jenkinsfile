#!groovy

pipeline {
    agent {
        label 'housekeeping'
    }
    options {
        timestamps()
        timeout(time: 2, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '1'))
        disableConcurrentBuilds()
    }
    parameters {
        string(name: 'FILE_NAME_TO_UPLOAD', defaultValue: 'Spectrecoin-Build241-81bf107f-???', description: 'File name to upload from Github to download area')
    }
    environment {
        // In case another branch beside master or develop should be deployed, enter it here
        BRANCH_TO_DEPLOY = 'automateWindowsInstallerCreation'
        DISCORD_WEBHOOK = credentials('DISCORD_WEBHOOK')
        GITHUB_CI_TOKEN = credentials('GITHUB_CI_TOKEN')
        DOMAIN = "download.spectreproject.io"
        FILE_NAME_TO_USE = sh(
                script: '''
                        echo ${FILE_NAME_TO_UPLOAD} | sed -E 's/\\.\\.+//g'
                    ''',
                returnStdout: true
        ).trim()
        GIT_TAG_TO_USE = sh(
                script: '''
                    echo ${FILE_NAME_TO_UPLOAD} | sed -E 's/\\.\\.+//g' | cut -d "-" -f2
                ''',
                returnStdout: true
        ).trim()
        GIT_COMMIT_SHORT = sh(
                script: '''
                    echo ${FILE_NAME_TO_UPLOAD} | sed -E 's/\\.\\.+//g' | cut -d "-" -f3
                ''',
                returnStdout: true
        ).trim()
    }
    stages {
        stage('Notification') {
            steps {
                // Using result state 'ABORTED' to mark the message on discord with a white border.
                // Makes it easier to distinguish job-start from job-finished
                discordSend(
                        description: "Started build #$env.BUILD_NUMBER",
                        image: '',
                        //link: "$env.BUILD_URL",
                        successful: true,
                        result: "ABORTED",
                        thumbnail: 'https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png',
                        title: "$env.JOB_NAME",
                        webhookURL: DISCORD_WEBHOOK
                )
            }
        }
        stage('Download') {
            steps {
                sshagent(['upload-to-download-site']) {
                    sh(
                            script: """
                                if curl --output /dev/null --silent --fail -r 0-0 "https://github.com/spectrecoin/spectre/releases/download/${GIT_TAG_TO_USE}/${FILE_NAME_TO_USE}" ; then
                                    sed -i "/download.spectreproject.io/d" ~/.ssh/known_hosts
                                    echo \$(ssh-keyscan ${DOMAIN},`nslookup ${DOMAIN} | awk '/^Address: / { print \$2 ; exit }'`) >> ~/.ssh/known_hosts
                                    ssh jenkins@download.spectreproject.io "mkdir -p /var/www/html/files/${GIT_TAG_TO_USE} \\
                                        && if [ -e /var/www/html/files/${GIT_TAG_TO_USE}/${FILE_NAME_TO_USE} ] ; then rm -f /var/www/html/files/${GIT_TAG_TO_USE}/${FILE_NAME_TO_USE} ; fi \\
                                        && cd /var/www/html/files/${GIT_TAG_TO_USE} \\
                                        && wget https://github.com/spectrecoin/spectre/releases/download/${GIT_TAG_TO_USE}/${FILE_NAME_TO_USE}"
                                else
                                    echo "File ${FILE_NAME_TO_USE} not found on Github"
                                fi
                            """
                    )
                }
            }
        }
    }
    post {
        success {
            script {
                if (!hudson.model.Result.SUCCESS.equals(currentBuild.getPreviousBuild()?.getResult())) {
                    emailext(
                            subject: "GREEN: '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                            body: '${JELLY_SCRIPT,template="html"}',
                            recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']],
//                            to: "to@be.defined",
//                            replyTo: "to@be.defined"
                    )
                }
                discordSend(
                        description: "Build #$env.BUILD_NUMBER successfully created and uploaded Spectrecoin bootstrap archive",
                        image: '',
                        //link: "$env.BUILD_URL",
                        successful: true,
                        thumbnail: 'https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png',
                        title: "$env.JOB_NAME",
                        webhookURL: DISCORD_WEBHOOK
                )
            }
        }
        unstable {
            emailext(
                    subject: "YELLOW: '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                    body: '${JELLY_SCRIPT,template="html"}',
                    recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']],
//                    to: "to@be.defined",
//                    replyTo: "to@be.defined"
            )
            discordSend(
                    description: "Build #$env.BUILD_NUMBER finished unstable",
                    image: '',
                    //link: "$env.BUILD_URL",
                    successful: true,
                    result: "UNSTABLE",
                    thumbnail: 'https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png',
                    title: "$env.JOB_NAME",
                    webhookURL: DISCORD_WEBHOOK
            )
        }
        failure {
            emailext(
                    subject: "RED: '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                    body: '${JELLY_SCRIPT,template="html"}',
                    recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']],
//                    to: "to@be.defined",
//                    replyTo: "to@be.defined"
            )
            discordSend(
                    description: "Build #$env.BUILD_NUMBER failed!",
                    image: '',
                    //link: "$env.BUILD_URL",
                    successful: false,
                    thumbnail: 'https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png',
                    title: "$env.JOB_NAME",
                    webhookURL: DISCORD_WEBHOOK
            )
        }
        aborted {
            discordSend(
                    description: "Build #$env.BUILD_NUMBER was aborted",
                    image: '',
                    //link: "$env.BUILD_URL",
                    successful: true,
                    result: "ABORTED",
                    thumbnail: 'https://wiki.jenkins-ci.org/download/attachments/2916393/headshot.png',
                    title: "$env.JOB_NAME",
                    webhookURL: DISCORD_WEBHOOK
            )
        }
    }
}
