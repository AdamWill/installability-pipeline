#!groovy

@Library('fedora-pipeline-library@prototype') _

import groovy.json.JsonOutput


def pipelineMetadata = [
    pipelineName: 'installability',
    pipelineDescription: 'Installability Test',
    testCategory: 'integration',
    testType: 'tier1',
    maintainer: 'Fedora CI',
    docs: 'https://github.com/fedora-ci/installability-pipeline',
    contact: [
        irc: '#fedora-ci',
        email: 'ci@lists.fedoraproject.org'
    ],
]
def artifactId
def testingFarmResult


pipeline {

    options {
        buildDiscarder(logRotator(numToKeepStr: '200'))
    }

    agent {
        label 'fedora-ci-agent'
    }

    parameters {
        string(name: 'ARTIFACT_ID', defaultValue: null, trim: true, description: '"koji-build:<taskId>" for Koji builds; Example: koji-build:42376994')
        string(name: 'ADDITIONAL_ARTIFACT_IDS', defaultValue: null, trim: true, description: 'A comma-separated list of additional ARTIFACT_IDs')
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    artifactId = params.ARTIFACT_ID

                    if (!artifactId) {
                        abort('ARTIFACT_ID is missing')
                    }
                }
                setBuildNameFromArtifactId(artifactId: artifactId)
                sendMessage(type: 'queued', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

        stage('Test') {
            steps {
                sendMessage(type: 'running', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())

                script {

                    def additionalArtifacts = []
                    if (params.ADDITIONAL_ARTIFACT_IDS) {
                        params.ADDITIONAL_ARTIFACT_IDS.split(',').each { additionalArtifactId ->
                            additionalArtifacts.add([id: "${additionalArtifactId.split(':')[1]}", type: 'fedora-koji-build'])
                        }
                    }

                    def requestPayload = """
                        {
                            "api_key": "xxx",
                            "test": {
                                "fmf": {
                                    "url": "${getGitUrl()}",
                                    "ref": "${getGitRef()}",
                                }
                            }
                            "environments": {
                                "arch": "x86_64",
                                "os": {
                                    "compose": "${getReleaseIdFromBranch}"
                                },
                                "variables": {
                                    "RELEASE_ID": "${getReleaseIdFromBranch()}",
                                    "TASK_ID": "${artifactId.split(':')[1]}"
                                },
                                "artifacts": "${JsonOutput.toJson(additionalArtifacts)}"
                            }
                        }
                    """
                    def response = submitTestingFarmRequest(payload: requestPayload)
                    testingFarmResult = waitForTestingFarmResults(requestId: response['id'], timeout: 30)

                    evaluateTestingFarmResults(testingFarmResult)
                }
            }
        }
    }

    post {
        always {
            echo 'No Testing Farm, nothing to archive.'
        }
        success {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
        }
        failure {
            sendMessage(type: 'error', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            echo """
*******************************************************************************************************************
Testing Farm is not up and running yet so this test will always fail. You can safely ignore it now. But stay tuned!
*******************************************************************************************************************
            """
        }
        unstable {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
        }
    }
}
