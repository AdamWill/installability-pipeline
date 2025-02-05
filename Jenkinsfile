#!groovy

retry (10) {
    // load pipeline configuration into the environment
    httpRequest("${FEDORA_CI_PIPELINES_CONFIG_URL}/environment").content.split('\n').each { l ->
        l = l.trim(); if (l && !l.startsWith('#')) { env["${l.split('=')[0].trim()}"] = "${l.split('=')[1].trim()}" }
    }
}

def pipelineMetadata = [
    pipelineName: 'installability',
    pipelineDescription: 'Try to install, upgrade, downgrade and remove RPM packages.',
    testCategory: 'functional',
    testType: 'installability',
    maintainer: 'Fedora CI',
    docs: 'https://github.com/fedora-ci/mini-tps',
    contact: [
        irc: '#fedora-ci',
        email: 'ci@lists.fedoraproject.org'
    ],
]
def artifactId
def additionalArtifactIds
def testingFarmRequestId
def testingFarmResult
def config
def ignoreList
def artifactName
def pipelineRepoUrlAndRef
def hook
def runUrl


pipeline {

    agent none

    libraries {
        lib("fedora-pipeline-library@${env.PIPELINE_LIBRARY_VERSION}")
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: env.DEFAULT_DAYS_TO_KEEP_LOGS, artifactNumToKeepStr: env.DEFAULT_ARTIFACTS_TO_KEEP))
        timeout(time: env.DEFAULT_PIPELINE_TIMEOUT_MINUTES, unit: 'MINUTES')
        skipDefaultCheckout(true)
    }

    parameters {
        string(name: 'ARTIFACT_ID', defaultValue: '', trim: true, description: '"koji-build:&lt;taskId&gt;" for Koji builds; Example: koji-build:46436038')
        string(name: 'ADDITIONAL_ARTIFACT_IDS', defaultValue: '', trim: true, description: 'A comma-separated list of additional ARTIFACT_IDs')
        string(name: 'TEST_PROFILE', defaultValue: env.FEDORA_CI_RAWHIDE_RELEASE_ID, trim: true, description: "A name of the test profile to use; Example: ${env.FEDORA_CI_RAWHIDE_RELEASE_ID}")
    }

    environment {
        TESTING_FARM_API_KEY = credentials('testing-farm-api-key')
    }

    stages {
        stage('Prepare') {
            agent {
                label pipelineMetadata.pipelineName
            }
            steps {
                script {
                    artifactId = params.ARTIFACT_ID
                    additionalArtifactIds = params.ADDITIONAL_ARTIFACT_IDS
                    artifactName = setBuildNameFromArtifactId(artifactId: artifactId, profile: params.TEST_PROFILE)

                    checkout scm
                    config = loadConfig(profile: params.TEST_PROFILE)
                    pipelineRepoUrlAndRef = [url: "${getGitUrl()}", ref: "${getGitRef()}"]

                    // check if the package is on the ignore list
                    ignoreList = loadConfig().get('ignore_list', [])
                    if (artifactName in ignoreList) {
                        abort("${artifactName} is on the pipeline ignore list, skipping...")
                    }

                    if (!artifactId) {
                        abort('ARTIFACT_ID is missing')
                    }
                }
                // sendMessage(type: 'queued', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

        stage('Schedule Test') {
            agent {
                label pipelineMetadata.pipelineName
            }
            steps {
                script {
                    def requestPayload = [
                        api_key: "${env.TESTING_FARM_API_KEY}",
                        test: [
                            fmf: pipelineRepoUrlAndRef
                        ],
                        environments: [
                            [
                                arch: "x86_64",
                                os: [ compose: "${config.compose}" ],
                                variables: [
                                    PROFILE_NAME: "${config.profile_name}",
                                    TASK_ID: "${getIdFromArtifactId(artifactId: artifactId)}",
                                    ADDITIONAL_TASK_IDS: "${getIdFromArtifactId(additionalArtifactIds: additionalArtifactIds, separator: ' ')}"
                                ]
                            ]
                        ]
                    ]
                    hook = registerWebhook()
                    requestPayload['notification'] = ['webhook': [url: hook.getURL()]]

                    def response = submitTestingFarmRequest(payloadMap: requestPayload)
                    testingFarmRequestId = response['id']
                }
                // sendMessage(type: 'running', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

        stage('Wait for Test Results') {
            agent none
            steps {
                script {
                    def response = waitForTestingFarm(requestId: testingFarmRequestId, hook: hook)
                    testingFarmResult = response.apiResponse
                    runUrl = "${FEDORA_CI_TESTING_FARM_ARTIFACTS_URL}/${testingFarmRequestId}"
                }
            }
        }
    }

    post {
        always {
            evaluateTestingFarmResults(testingFarmResult)
        }
        success {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, runUrl: runUrl, dryRun: isPullRequest())
        }
        failure {
            sendMessage(type: 'error', artifactId: artifactId, pipelineMetadata: pipelineMetadata, runUrl: runUrl, dryRun: isPullRequest())
        }
        unstable {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, runUrl: runUrl, dryRun: isPullRequest())
        }
        aborted {
            script {
                if (artifactId && !testingFarmRequestId) {
                    sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, isSkipped: true, note: "${env.ABORT_MESSAGE}", dryRun: isPullRequest())
                }
                if (isTimeoutAborted(timeout: env.DEFAULT_PIPELINE_TIMEOUT_MINUTES, unit: 'MINUTES')) {
                    sendMessage(type: 'error', artifactId: artifactId, errorReason: 'Timeout has been exceeded, pipeline aborted.', pipelineMetadata: pipelineMetadata, runUrl: runUrl, dryRun: isPullRequest())
                }
            }
        }
    }
}
