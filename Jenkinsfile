pipeline {
    agent any

    environment {
        BITBUCKET_USERNAME = 'akshaysunil201'
        BITBUCKET_APP_PASSWORD = credentials('bit_bucket_token')
        GITHUB_USERNAME = 'akshayviola'
        GITHUB_TOKEN = credentials('github_token')
    }

    stages {
        stage('Sync Repositories') {
            steps {
                script {
                    // Prompt for source and destination repository details
                    def sourcePlatform = input message: 'Enter the source platform (Bitbucket or GitHub):', parameters: [string(defaultValue: '', description: 'Source platform', name: 'SOURCE_PLATFORM')]
                    def sourceWorkspaceOrUsername = input message: 'Enter the source workspace (for Bitbucket) or username (for GitHub):', parameters: [string(defaultValue: '', description: 'Source workspace/username', name: 'SOURCE_WORKSPACE_OR_USERNAME')]
                    def destPlatform = input message: 'Enter the destination platform (Bitbucket or GitHub):', parameters: [string(defaultValue: '', description: 'Destination platform', name: 'DEST_PLATFORM')]
                    def destWorkspaceOrUsername = input message: 'Enter the destination workspace (for Bitbucket) or username (for GitHub):', parameters: [string(defaultValue: '', description: 'Destination workspace/username', name: 'DEST_WORKSPACE_OR_USERNAME')]
                    def repoName = input message: 'Enter the repository name:', parameters: [string(defaultValue: '', description: 'Repository name', name: 'REPO_NAME')]
                    def branchesInput = input message: 'Enter the branches to sync (comma-separated, e.g., main,develop):', parameters: [string(defaultValue: '', description: 'Branches to sync', name: 'BRANCHES_INPUT')]

                    def branchList = branchesInput.split(',')

                    def checkGithubRepo = { String username, String repoToCheck ->
                        def repoCheck = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" -u \"${env.GITHUB_USERNAME}:${env.GITHUB_TOKEN}\" \"https://api.github.com/repos/${username}/${repoToCheck}\"", returnStdout: true).trim()
                        return repoCheck == '200'
                    }

                    def checkBitbucketRepo = { String workspace, String repoToCheck ->
                        def response = sh(script: "curl -s -u \"${env.BITBUCKET_USERNAME}:${env.BITBUCKET_APP_PASSWORD}\" \"https://api.bitbucket.org/2.0/repositories/${workspace}/${repoToCheck}\"", returnStdout: true).trim()
                        return response.contains("\"name\": \"${repoToCheck}\"")
                    }

                    def createGithubRepo = { String username, String repoToCreate ->
                        sh(script: "curl -s -X POST -u \"${env.GITHUB_USERNAME}:${env.GITHUB_TOKEN}\" -d '{\"name\":\"${repoToCreate}\"}' https://api.github.com/user/repos")
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
                        if (!checkGithubRepo(sourceWorkspaceOrUsername, repoName)) {
                            error "Repository '${repoName}' does not exist on GitHub."
                        }
                        sourceRepoUrl = "https://${env.GITHUB_USERNAME}:${env.GITHUB_TOKEN}@github.com/${sourceWorkspaceOrUsername}/${repoName}.git"
                    } else {
                        error "Unsupported source platform: ${sourcePlatform}. Please choose either 'Bitbucket' or 'GitHub'."
                    }

                    if (destPlatform.toLowerCase() == 'bitbucket') {
                        if (!checkBitbucketRepo(destWorkspaceOrUsername, repoName)) {
                            createBitbucketRepo(destWorkspaceOrUsername, repoName) // Create the repo if it doesn't exist
                        }
                        destRepoUrl = "https://${env.BITBUCKET_USERNAME}:${env.BITBUCKET_APP_PASSWORD}@bitbucket.org/${destWorkspaceOrUsername}/${repoName}.git"
                    } else if (destPlatform.toLowerCase() == 'github') {
                        if (!checkGithubRepo(destWorkspaceOrUsername, repoName)) {
                            createGithubRepo(destWorkspaceOrUsername, repoName) // Create the repo if it doesn't exist
                        }
                        destRepoUrl = "https://${env.GITHUB_USERNAME}:${env.GITHUB_TOKEN}@github.com/${destWorkspaceOrUsername}/${repoName}.git"
                    } else {
                        error "Unsupported destination platform: ${destPlatform}. Please choose either 'Bitbucket' or 'GitHub'."
                    }

                    syncBranches(sourceRepoUrl, destRepoUrl, branchList)
                }
            }
        }
    }
}
