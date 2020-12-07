#!groovy

@Library('fedora-pipeline-library@candidate2') _

def pipelineMetadata = [
    pipelineName: 'compose-ci',
    pipelineDescription: 'Run ODCS compose on RPM builds',
    testCategory: 'static-analysis',
    testType: 'compose-ci',
    maintainer: 'Fedora CI',
    docs: 'https://pagure.io/odcs/',
    contact: [
        irc: '#fedora-ci',
        email: 'ci@lists.fedoraproject.org'
    ],
]
def artifactId
def xunit

def podYAML = """
spec:
  containers:
  - name: odcs
    image: quay.io/jkaluza/odcs:latest
    tty: true
"""

pipeline {

    agent {
        kubernetes {
            yaml podYAML
            defaultContainer 'odcs'
        }
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '45', artifactNumToKeepStr: '100'))
        timeout(time: 12, unit: 'HOURS')
    }

    parameters {
        string(name: 'ARTIFACT_ID', defaultValue: '', trim: true, description: '"koji-build:&lt;taskId&gt;" for Koji builds; Example: koji-build:46436038')
    }

    environment {
        ODCS_API_KEY = credentials('odcs-api-key')
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    artifactId = params.ARTIFACT_ID
                    setBuildNameFromArtifactId(artifactId: artifactId)

                    if (!artifactId) {
                        abort('ARTIFACT_ID is missing')
                    }
                }
                sendMessage(type: 'queued', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

        stage('Execute compose') {
            steps {
                script {
                    run_compose_status = sh(returnStatus: true, script: """
                    ./scripts/run-compose --compose-name releng_compose_ci --compose-branch master --artifact-id ${artifactId}
                    """)
                    assert run_compose_status == 0 : "Compose build failed"
                }
                sendMessage(type: 'running', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

//         stage('Process Test Results (XUnit)') {
//             when {
//                 expression { xunit }
//             }
//             agent {
//                 kubernetes {
//                     yaml podYAML
//                     defaultContainer 'pipeline-agent'
//                 }
//             }
//             steps {
//                 script {
//                     // Convert Testing Farm XUnit into JUnit and store the result in Jenkins
//                     writeFile file: 'tfxunit.xml', text: "${xunit}"
//                     sh script: "tfxunit2junit --docs-url ${pipelineMetadata['docs']} tfxunit.xml > xunit.xml"
//                     junit(allowEmptyResults: true, keepLongStdio: true, testResults: 'xunit.xml')
//                 }
//             }
//         }
    }

    post {
        success {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, xunit: xunit, dryRun: isPullRequest())
        }
        failure {
            sendMessage(type: 'error', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
        }
        unstable {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, xunit: xunit, dryRun: isPullRequest())
        }
    }
}
