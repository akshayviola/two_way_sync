pipeline {
    agent any

    parameters {
        choice(name: 'SOURCE_PLATFORM', choices: ['Bitbucket', 'GitHub'], description: 'Choose the source platform')
        string(name: 'SOURCE_WORKSPACE_OR_USERNAME', defaultValue: '', description: 'Enter the source workspace (for Bitbucket) or username (for GitHub)')
        string(name: 'SOURCE_TOKEN_ID', defaultValue: '', description: 'Jenkins Credentials ID for Source Token')
        
        choice(name: 'DEST_PLATFORM', choices: ['Bitbucket', 'GitHub'], description: 'Choose the destination platform')
        string(name: 'DEST_WORKSPACE_OR_USERNAME', defaultValue: '', description: 'Enter the destination workspace (for Bitbucket) or username (for GitHub)')
        string(name: 'DEST_TOKEN_ID', defaultValue: '', description: 'Jenkins Credentials ID for Destination Token')
        
        string(name: 'REPO_NAME', defaultValue: '', description: 'Enter the repository name')
        string(name: 'BRANCHES_INPUT', defaultValue: '', description: 'Branches to sync (comma-separated, e.g., main,develop)')
    }

    stages {
        stage('Sync Repositories') {
            steps {
                script {
                    def branchList = params.BRANCHES_INPUT.split(',')
                    
                    // Function to check if a GitHub repository exists
                    def checkGithubRepo = { String username, String repoToCheck, String githubToken ->
                        def repoCheck = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" -u \"${username}:${githubToken}\" \"https://api.github.com/repos/${username}/${repoToCheck}\"", returnStdout: true).trim()
                        return repoCheck == '200'
                    }

                    // Function to check if a Bitbucket repository exists
                    def checkBitbucketRepo = { String workspace, String repoToCheck, String bitbucketToken ->
                        def response = sh(script: "curl -s -u \"${workspace}:${bitbucketToken}\" \"https://api.bitbucket.org/2.0/repositories/${workspace}/${repoToCheck}\"", returnStdout: true).trim()
                        return response.contains("\"name\": \"${repoToCheck}\"")
                    }

                    // Function to create a GitHub repository
                    def createGithubRepo = { String username, String repoToCreate, String githubToken ->
                        sh(script: "curl -s -X POST -u \"${username}:${githubToken}\" -d '{\"name\":\"${repoToCreate}\"}' https://api.github.com/user/repos")
                    }

                    // Function to create a Bitbucket repository
                    def createBitbucketRepo = { String workspace, String repoToCreate, String bitbucketToken ->
                        sh(script: "curl -s -X POST -u \"${workspace}:${bitbucketToken}\" -H \"Content-Type: application/json\" -d '{\"scm\": \"git\", \"name\": \"${repoToCreate}\"}' https://api.bitbucket.org/2.0/repositories/${workspace}/${repoToCreate}")
                    }

                    // Function to sync branches between repositories
                    def syncBranches = { String sourceRepoUrl, String destRepoUrl, List branchListToSync ->
                        branchListToSync.each { branch ->
                            echo "Syncing branch '${branch}' for repository '${params.REPO_NAME}' from ${params.SOURCE_PLATFORM} to ${params.DEST_PLATFORM}..."
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
                        echo "Repository '${params.REPO_NAME}' branches have been synced successfully from ${params.SOURCE_PLATFORM} to ${params.DEST_PLATFORM}."
                    }

                    def sourceRepoUrl, destRepoUrl

                    // Use withCredentials to retrieve the tokens based on platform
                    withCredentials([string(credentialsId: params.SOURCE_TOKEN_ID, variable: 'SOURCE_TOKEN'),
                                     string(credentialsId: params.DEST_TOKEN_ID, variable: 'DEST_TOKEN')]) {

                        if (params.SOURCE_PLATFORM.toLowerCase() == 'bitbucket') {
                            if (!checkBitbucketRepo(params.SOURCE_WORKSPACE_OR_USERNAME, params.REPO_NAME, SOURCE_TOKEN)) {
                                error "Repository '${params.REPO_NAME}' does not exist on Bitbucket."
                            }
                            sourceRepoUrl = "https://${params.SOURCE_WORKSPACE_OR_USERNAME}:${SOURCE_TOKEN}@bitbucket.org/${params.SOURCE_WORKSPACE_OR_USERNAME}/${params.REPO_NAME}.git"
                        } else if (params.SOURCE_PLATFORM.toLowerCase() == 'github') {
                            if (!checkGithubRepo(params.SOURCE_WORKSPACE_OR_USERNAME, params.REPO_NAME, SOURCE_TOKEN)) {
                                error "Repository '${params.REPO_NAME}' does not exist on GitHub."
                            }
                            sourceRepoUrl = "https://${params.SOURCE_WORKSPACE_OR_USERNAME}:${SOURCE_TOKEN}@github.com/${params.SOURCE_WORKSPACE_OR_USERNAME}/${params.REPO_NAME}.git"
                        } else {
                            error "Unsupported source platform: ${params.SOURCE_PLATFORM}. Please choose either 'Bitbucket' or 'GitHub'."
                        }

                        if (params.DEST_PLATFORM.toLowerCase() == 'bitbucket') {
                            if (!checkBitbucketRepo(params.DEST_WORKSPACE_OR_USERNAME, params.REPO_NAME, DEST_TOKEN)) {
                                createBitbucketRepo(params.DEST_WORKSPACE_OR_USERNAME, params.REPO_NAME, DEST_TOKEN) // Create the repo if it doesn't exist
                            }
                            destRepoUrl = "https://${params.DEST_WORKSPACE_OR_USERNAME}:${DEST_TOKEN}@bitbucket.org/${params.DEST_WORKSPACE_OR_USERNAME}/${params.REPO_NAME}.git"
                        } else if (params.DEST_PLATFORM.toLowerCase() == 'github') {
                            if (!checkGithubRepo(params.DEST_WORKSPACE_OR_USERNAME, params.REPO_NAME, DEST_TOKEN)) {
                                createGithubRepo(params.DEST_WORKSPACE_OR_USERNAME, params.REPO_NAME, DEST_TOKEN) // Create the repo if it doesn't exist
                            }
                            destRepoUrl = "https://${params.DEST_WORKSPACE_OR_USERNAME}:${DEST_TOKEN}@github.com/${params.DEST_WORKSPACE_OR_USERNAME}/${params.REPO_NAME}.git"
                        } else {
                            error "Unsupported destination platform: ${params.DEST_PLATFORM}. Please choose either 'Bitbucket' or 'GitHub'."
                        }

                        // Sync branches
                        syncBranches(sourceRepoUrl, destRepoUrl, branchList)
                    }
                }
            }
        }
    }
}
