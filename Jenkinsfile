pipeline {
    agent any

    parameters {
        string(name: 'SOURCE_PLATFORM', defaultValue: '', description: 'Source platform (Bitbucket or GitHub)')
        string(name: 'SOURCE_WORKSPACE_OR_USERNAME', defaultValue: '', description: 'Source workspace (for Bitbucket) or username (for GitHub)')
        string(name: 'DEST_PLATFORM', defaultValue: '', description: 'Destination platform (Bitbucket or GitHub)')
        string(name: 'DEST_WORKSPACE_OR_USERNAME', defaultValue: '', description: 'Destination workspace (for Bitbucket) or username (for GitHub)')
        string(name: 'REPO_NAME', defaultValue: '', description: 'Repository name')
        string(name: 'BRANCHES_INPUT', defaultValue: '', description: 'Branches to sync (comma-separated, e.g., main,develop)')
        string(name: 'SOURCE_GITHUB_TOKEN_ID', defaultValue: '', description: 'Credential ID for the source GitHub token')
        string(name: 'DEST_GITHUB_TOKEN_ID', defaultValue: '', description: 'Credential ID for the destination GitHub token')
    }

    environment {
        BITBUCKET_USERNAME = 'akshaysunil201'
        BITBUCKET_APP_PASSWORD = credentials('bit_bucket_token')
    }

    stages {
        stage('Sync Repositories') {
            steps {
                script {
                    // Retrieve parameters
                    def sourcePlatform = params.SOURCE_PLATFORM
                    def sourceWorkspaceOrUsername = params.SOURCE_WORKSPACE_OR_USERNAME
                    def destPlatform = params.DEST_PLATFORM
                    def destWorkspaceOrUsername = params.DEST_WORKSPACE_OR_USERNAME
                    def repoName = params.REPO_NAME
                    def branchesInput = params.BRANCHES_INPUT

                    def branchList = branchesInput.split(',')

                    // Retrieve GitHub tokens
                    withCredentials([string(credentialsId: params.SOURCE_GITHUB_TOKEN_ID, variable: 'SOURCE_GITHUB_TOKEN'),
                                     string(credentialsId: params.DEST_GITHUB_TOKEN_ID, variable: 'DEST_GITHUB_TOKEN')]) {

                        def checkGithubRepo = { String username, String repoToCheck, String token ->
                            def repoCheck = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" -u \"${username}:${token}\" \"https://api.github.com/repos/${username}/${repoToCheck}\"", returnStdout: true).trim()
                            return repoCheck == '200'
                        }

                        def checkBitbucketRepo = { String workspace, String repoToCheck ->
                            def response = sh(script: "curl -s -u \"${env.BITBUCKET_USERNAME}:${env.BITBUCKET_APP_PASSWORD}\" \"https://api.bitbucket.org/2.0/repositories/${workspace}/${repoToCheck}\"", returnStdout: true).trim()
                            return response.contains("\"name\": \"${repoToCheck}\"")
                        }

                        def createGithubRepo = { String username, String repoToCreate, String token ->
                            sh(script: "curl -s -X POST -u \"${username}:${token}\" -d '{\"name\":\"${repoToCreate}\"}' https://api.github.com/user/repos")
                        }

                        def createBitbucketRepo = { String workspace, String repoToCreate ->
                            sh(script: "curl -s -X POST -u \"${env.BITBUCKET_USERNAME}:${env.BITBUCKET_APP_PASSWORD}\" -H \"Content-Type: application/json\" -d '{\"scm\": \"git\", \"name\": \"${repoToCreate}\"}' https://api.bitbucket.org/2.0/repositories/${workspace}/${repoToCreate}")
                        }

                        def syncBranches = { String sourceRepoUrl, String destRepoUrl, List branchListToSync ->
                            branchListToSync.each { branch ->
                                echo "Syncing branch '${branch}' for repository '${repoName}' from ${sourcePlatform} to ${destPlatform}..."
                                def tmpDir = sh(script: 'mktemp -d', returnStdout: true).trim()
                                try {
                                    sh(script: "git clone --bare \"${sourceRepoUrl}\" \"${tmpDir}/source-repo\"")
                                    dir("${tmpDir}/source-repo") {
                                        sh(script: "git fetch origin ${branch}")
                                    }
                                    sh(script: "git -C \"${tmpDir}/source-repo\" push \"${destRepoUrl}\" ${branch}")
                                } finally {
                                    sh(script: "rm -rf \"${tmpDir}\"")
                                }
                            }
                            echo "Repository '${repoName}' branches have been synced successfully from ${sourcePlatform} to ${destPlatform}."
                        }

                        def sourceRepoUrl, destRepoUrl

                        if (sourcePlatform.toLowerCase() == 'bitbucket') {
                            if (!checkBitbucketRepo(sourceWorkspaceOrUsername, repoName)) {
                                error "Repository '${repoName}' does not exist on Bitbucket."
                            }
                            sourceRepoUrl = "https://${env.BITBUCKET_USERNAME}:${env.BITBUCKET_APP_PASSWORD}@bitbucket.org/${sourceWorkspaceOrUsername}/${repoName}.git"
                        } else if (sourcePlatform.toLowerCase() == 'github') {
                            if (!checkGithubRepo(sourceWorkspaceOrUsername, repoName, env.SOURCE_GITHUB_TOKEN)) {
                                error "Repository '${repoName}' does not exist on GitHub."
                            }
                            sourceRepoUrl = "https://${sourceWorkspaceOrUsername}:${env.SOURCE_GITHUB_TOKEN}@github.com/${sourceWorkspaceOrUsername}/${repoName}.git"
                        } else {
                            error "Unsupported source platform: ${sourcePlatform}. Please choose either 'Bitbucket' or 'GitHub'."
                        }

                        if (destPlatform.toLowerCase() == 'bitbucket') {
                            if (!checkBitbucketRepo(destWorkspaceOrUsername, repoName)) {
                                createBitbucketRepo(destWorkspaceOrUsername, repoName) // Create the repo if it doesn't exist
                            }
                            destRepoUrl = "https://${env.BITBUCKET_USERNAME}:${env.BITBUCKET_APP_PASSWORD}@bitbucket.org/${destWorkspaceOrUsername}/${repoName}.git"
                        } else if (destPlatform.toLowerCase() == 'github') {
                            if (!checkGithubRepo(destWorkspaceOrUsername, repoName, env.DEST_GITHUB_TOKEN)) {
                                createGithubRepo(destWorkspaceOrUsername, repoName, env.DEST_GITHUB_TOKEN) // Create the repo if it doesn't exist
                            }
                            destRepoUrl = "https://${destWorkspaceOrUsername}:${env.DEST_GITHUB_TOKEN}@github.com/${destWorkspaceOrUsername}/${repoName}.git"
                        } else {
                            error "Unsupported destination platform: ${destPlatform}. Please choose either 'Bitbucket' or 'GitHub'."
                        }

                        syncBranches(sourceRepoUrl, destRepoUrl, branchList)
                    }
                }
            }
        }
    }
}
