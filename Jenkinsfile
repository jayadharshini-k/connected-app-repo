/* groovylint-disable GStringExpressionWithinString, LineLength, NestedBlockDepth, NoDef, VariableTypeRequired */
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
                    /* groovylint-disable-next-line DuplicateStringLiteral, NestedBlockDepth */
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
    }

        stage('Update Project Version') {
            steps {
                script {
                    /* groovylint-disable-next-line DuplicateStringLiteral */
                    dir('D:\\workspace9') {
                        /* groovylint-disable-next-line NoDef, VariableTypeRequired */
                        def buildNumber = env.BUILD_NUMBER ?: '0'
                        /* groovylint-disable-next-line NoDef, VariableTypeRequired */
                        def newVersion = "9.1.${buildNumber}"
                        bat "mvn versions:set -DnewVersion=${newVersion}"

                        // Create a version-based directory to store JAR files
                        def versionDir = "jars_${newVersion}"
                        bat "mkdir ${versionDir}"

                        // Create a JAR with the new version and move it to the version-based directory
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
                def copiedPomPath // Declare the variable at the higher scope

                try {
                    // Extract username and password from combined credentials
                    def creds = CLOUDHUB_CREDENTIAL.split(':')
                    def username = creds[0]
                    def password = creds[1]

                    // Read the original pom.xml content
                    def originalPomContent = readFile("${WORKSPACE_PATH}\\pom.xml")

                    // Create a copy of the original pom.xml
                    copiedPomPath = "${WORKSPACE_PATH}\\copied-pom.xml"
                    writeFile(file: copiedPomPath, text: originalPomContent)

                    // Print non-secret values for debugging
                    echo "DEPLOY_ENVIRONMENT: ${DEPLOY_ENVIRONMENT.toString()}"

                    // Replace placeholders directly
                    def modifiedPomContent = originalPomContent.replace('${deploy.environment}', DEPLOY_ENVIRONMENT)
                                                            .replace('${cloudhub.username}', username)
                                                            .replace('${cloudhub.password}', password)
                                                            .replace('${BUSINESS_GROUP_ID}', BUSINESS_GROUP_ID)
                                                            .replace('${deploy.environment}-app', "${DEPLOY_ENVIRONMENT}-app")

                    // Write the modified content to the copied pom.xml file
                    writeFile(file: copiedPomPath, text: modifiedPomContent)

                    // Navigate to the Git repository directory
                    dir("${WORKSPACE_PATH}") {
                        // Check if the Git repository is initialized
                        if (fileExists('.git')) {
                            // Build and deploy the project using the copied pom.xml
                            bat "mvn clean deploy -DmuleDeploy -P${DEPLOY_ENVIRONMENT} -X -f ${copiedPomPath}"

                            // Use Git Changelog plugin to generate changelog
                            def changelog = changelogGit(
                                from: 'origin/master',
                                to: 'HEAD',
                                format: 'markdown',
                                onlyCommitters: true
                            )

                            // Debugging: Print the changelog before sending the email
                            echo "Changelog:\n${changelog}"

                            // Send email notification for successful build with changelog
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
                    // Debugging: Print the exception details for troubleshooting
                    echo "Error: ${e}"

                    currentBuild.result = 'FAILURE'
                    throw e
                } finally {
                    if (copiedPomPath) {
                        // Delete the copied pom.xml after deployment
                        bat "del ${copiedPomPath}"
                    }
                }
            }
        }
    }
}
