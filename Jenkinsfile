/* groovylint-disable CompileStatic, DuplicateStringLiteral, GStringExpressionWithinString, NestedBlockDepth, NoDef, VariableTypeRequired */
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
                    // Extract username and password from combined credentials
                    def creds = CLOUDHUB_CREDENTIAL.split(':')
                    def username = creds[0]
                    def password = creds[1]

                    // Read the original pom.xml content
                    def originalPomContent = readFile("${WORKSPACE_PATH}\\pom.xml")

                    // Create a copy of the original pom.xml
                    def copiedPomPath = "${WORKSPACE_PATH}\\copied-pom.xml"
                    writeFile(file: copiedPomPath, text: originalPomContent)

                    try {
                        // Print non-secret values for debugging
                        echo "DEPLOY_ENVIRONMENT: ${DEPLOY_ENVIRONMENT}"

                        // Replace placeholders directly
                        def modifiedPomContent = originalPomContent.replace('${deploy.environment}', DEPLOY_ENVIRONMENT)
                                                                      .replace('${cloudhub.username}', username)
                                                                      .replace('${cloudhub.password}', password)
                                                                      .replace('${env.BUSINESS_GROUP_ID}', BUSINESS_GROUP_ID)
                                                                      .replace('${deploy.environment}-app', "${DEPLOY_ENVIRONMENT}-app")

                        // Write the modified content to the copied pom.xml file
                        writeFile(file: copiedPomPath, text: modifiedPomContent)
                        
                        // Navigate to the Git repository directory
                        dir("${WORKSPACE_PATH}") {
                            // Build and deploy the project using the copied pom.xml
                            bat "mvn clean deploy -DmuleDeploy -P${DEPLOY_ENVIRONMENT} -X -f ${copiedPomPath}"
                        }

                        // Get the changelog
                        def changelog = bat(script: 'git log --pretty=oneline origin/master..HEAD', returnStatus: true).text
                        echo "Changelog:\n${changelog}"

                        // Send email notification for successful build with changelog
                        emailext body: "The pipeline ${currentBuild.fullDisplayName} has succeeded.\nChangelog:\n${changelog}",
                                 subject: "Pipeline Succeeded: ${currentBuild.fullDisplayName}",
                                 mimeType: 'text/plain',
                                 to: 'jayadharshini.azuredevops@gmail.com',
                                 attachLog: true
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        throw e
                    } finally {
                        // Delete the copied pom.xml after deployment
                        bat "del ${copiedPomPath}"
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
                     attachLog: true  // Attach build log
        }
    }
}
