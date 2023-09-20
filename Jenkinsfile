pipeline {
    agent any

    environment {
        CONNECTED_APP_CREDENTIAL = credentials('3f28a56e-de58-498c-9edc-d98cb8d9838d')
        BUSINESS_GROUP_ID = credentials('MY_BUSINESS_GROUP_ID_SECRET')
        DEPLOY_ENVIRONMENT = 'dev' // Set the environment here
        WORKSPACE_PATH = 'D:\\connected-app'
    }

    stages {
        stage('Pulling latest code') {
            steps {
                script {
                    dir('D:\\connected-app') {
                        if (!fileExists('.git')) {
                            bat 'git init'
                            bat 'git remote add origin https://github.com/jayadharshini-k/connected-app-repo.git'
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
                    dir('D:\\connected-app') {
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
                    dir('D:\\connected-app') {
                        bat 'mvn deploy'
                    }
                }
            }
        }

        stage('Dev - Deploy') {
            steps {
                script {
                    // Extract username and password from combined credentials
                    def creds = CONNECTED_APP_CREDENTIAL.split(':')
                    def client_id = creds[0]
                    def client_secret = creds[1]

                    // Read the original pom.xml content
                    def originalPomContent = readFile("${WORKSPACE_PATH}\\pom.xml")

                    // Replace placeholders directly
                    def modifiedPomContent = originalPomContent.replaceAll('CONN_APP_CLIENT_ID', client_id)
                                                              .replaceAll('CONN_APP_CLIENT_SECRET', client_secret)
                                                              .replaceAll('BUSINESS_GROUP_ID', BUSINESS_GROUP_ID)
                                                              .replaceAll('dev', DEPLOY_ENVIRONMENT)
                                                              .replaceAll('${deploy.environment}-app', "${DEPLOY_ENVIRONMENT}-app")

                    // Write the modified content to the copied pom.xml file
                    def copiedPomPath = "${WORKSPACE_PATH}\\copied-pom.xml"
                    writeFile(file: copiedPomPath, text: modifiedPomContent)

                    try {
                        // Print non-secret values for debugging
                        echo "DEPLOY_ENVIRONMENT: ${DEPLOY_ENVIRONMENT.toString()}"

                        // Navigate to the Git repository directory
                        dir("${WORKSPACE_PATH}") {
                            // Check if the Git repository is initialized
                            if (fileExists('.git')) {
                                // Build and deploy the project using the copied pom.xml
                                bat "mvn clean deploy -DmuleDeploy -P${DEPLOY_ENVIRONMENT} -X -f ${copiedPomPath}"

                                // Retrieve change logs using git log and capture the output
                                def changelog = bat(script: 'git log origin/master', returnStdout: true).trim()

                                // Write changelog content to a file
                                writeFile(file: "${WORKSPACE_PATH}\\changelog-dev.txt", text: changelog)

                                // Send email notification for successful build with changelog attached
                                emailext body: "The pipeline ${currentBuild.fullDisplayName} has succeeded.\n",
                                         subject: "Pipeline Succeeded: ${currentBuild.fullDisplayName}",
                                         mimeType: 'text/plain',
                                         to: 'jayadharshini.azuredevops@gmail.com',
                                         attachmentsPattern: '**/changelog-dev.txt'

                            }
                        }
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
