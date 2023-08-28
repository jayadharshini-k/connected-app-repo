/* groovylint-disable CatchException, DuplicateStringLiteral, GStringExpressionWithinString, InvertedIfElse, LineLength, NestedBlockDepth, NoDef, VariableTypeRequired */
pipeline {
    agent any

    environment {
        CLOUDHUB_CREDENTIAL = credentials('a0f0c641-8967-43ea-a26c-21cbee9dc501')
        BUSINESS_GROUP_ID = credentials('MY_BUSINESS_GROUP_ID_SECRET')
        DEPLOY_ENVIRONMENT = 'dev' // Set the environment here
        WORKSPACE_PATH = 'D:\\workspace9'
    }

    stages {
        stage('Pulling latest code') {
            steps {
                script {
                    dir('D:\\workspace9') {
                        if (!fileExists('.git')) {
                            bat 'git init'
                            bat 'git remote add origin https://github.com/jayadharshini-k/final-mule.git'
                            bat 'git fetch origin master'
                        } else {
                            echo 'Git repository already initialized.'
                        }
                        bat 'git pull origin master'
                    }
                }
            }
        }

        stage('Update Project Version') {
            steps {
                script {
                    dir('D:\\workspace9') {
                        def buildNumber = env.BUILD_NUMBER ?: '0'
                        def newVersion = "9.1.${buildNumber}"
                        bat "mvn versions:set -DnewVersion=${newVersion}"

                        def versionDir = "jars_${newVersion}"
                        bat "mkdir ${versionDir}"

                        bat 'mvn clean package'
                        bat "move target\\*.jar ${versionDir}"
                    }
                }
            }
        }

        stage('Publish Assets to Exchange') {
            steps {
                script {
                    dir('D:\\workspace9') {
                        bat 'mvn deploy'
                    }
                }
            }
        }

        stage('Dev - Deploy') {
            steps {
                script {
                    def copiedPomPath

                    try {
                        def creds = CLOUDHUB_CREDENTIAL.split(':')
                        def username = creds[0]
                        def password = creds[1]

                        def originalPomContent = readFile("${WORKSPACE_PATH}\\pom.xml")

                        copiedPomPath = "${WORKSPACE_PATH}\\copied-pom.xml"
                        writeFile(file: copiedPomPath, text: originalPomContent)

                        echo "DEPLOY_ENVIRONMENT: ${DEPLOY_ENVIRONMENT}"

                        def modifiedPomContent = originalPomContent.replace('${deploy.environment}', DEPLOY_ENVIRONMENT)
                                                                  .replace('${cloudhub.username}', username)
                                                                  .replace('${cloudhub.password}', password)
                                                                  .replace('${BUSINESS_GROUP_ID}', BUSINESS_GROUP_ID)
                                                                  .replace('${deploy.environment}-app', "${DEPLOY_ENVIRONMENT}-app")

                        writeFile(file: copiedPomPath, text: modifiedPomContent)

                        dir("${WORKSPACE_PATH}") {
                            if (fileExists('.git')) {
                                bat "mvn clean deploy -DmuleDeploy -P${DEPLOY_ENVIRONMENT} -X -f ${copiedPomPath}"

                                def changelog = changelogGit(
                                    from: 'origin/master',
                                    to: 'HEAD',
                                    format: 'markdown',
                                    onlyCommitters: true
                                )

                                echo "Changelog:\n${changelog}"

                                emailext body: "The pipeline ${currentBuild.fullDisplayName} has succeeded.\nChangelog:\n${changelog}",
                                         subject: "Pipeline Succeeded: ${currentBuild.fullDisplayName}",
                                         mimeType: 'text/plain',
                                         to: 'jayadharshini.azuredevops@gmail.com',
                                         attachLog: true
                            } else {
                                echo 'Not a Git repository. Skipping deployment and changelog retrieval.'
                            }
                        }
                    } catch (Exception e) {
                        echo "Error: ${e}"

                        currentBuild.result = 'FAILURE'
                        throw e
                    } finally {
                        if (copiedPomPath) {
                            bat "del ${copiedPomPath}"
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            emailext body: "The pipeline ${currentBuild.fullDisplayName} has failed. Please find the logs attached.",
                     subject: "Pipeline Failed: ${currentBuild.fullDisplayName}",
                     mimeType: 'text/plain',
                     to: 'jayadharshini.azuredevops@gmail.com',
                     attachLog: true
        }
    }
}
