pipeline {
    agent any

    parameters {
        string(name: 'SOURCE_PLATFORM', defaultValue: '', description: 'Source platform (bitbucket/github)')
        string(name: 'DEST_PLATFORM', defaultValue: '', description: 'Destination platform (bitbucket/github)')
        string(name: 'SOURCE_USERNAME', defaultValue: '', description: 'Source username')
        string(name: 'DEST_USERNAME', defaultValue: '', description: 'Destination username')
        string(name: 'SOURCE_WORKSPACE', defaultValue: '', description: 'Source workspace or organization')
        string(name: 'DEST_ORGANIZATION', defaultValue: '', description: 'Destination organization or user account')
        booleanParam(name: 'USE_SSH', defaultValue: false, description: 'Use SSH for cloning and pushing?')
        password(name: 'SOURCE_TOKEN', defaultValue: '', description: 'Token for source platform')
        password(name: 'DEST_TOKEN', defaultValue: '', description: 'Token for destination platform')
    }

    stages {
        stage('Sync Repositories') {
            steps {
                script {
                    // Masking the tokens using the 'withCredentials' directive
                    withCredentials([string(credentialsId: 'source-token', variable: 'SOURCE_TOKEN'),
                                     string(credentialsId: 'dest-token', variable: 'DEST_TOKEN')]) {
                        def createRepoUrl = { platform, workspace, repo, useSsh, username, token ->
                            if (useSsh) {
                                return platform == 'bitbucket' ? "git@bitbucket.org:${workspace}/${repo}.git" : "git@github.com:${workspace}/${repo}.git"
                            } else {
                                return platform == 'bitbucket' ? "https://${username}:${token}@bitbucket.org/${workspace}/${repo}.git" : "https://${username}:${token}@github.com/${workspace}/${repo}.git"
                            }
                        }

                        def sourcePlatform = params.SOURCE_PLATFORM.toLowerCase().trim()
                        def destPlatform = params.DEST_PLATFORM.toLowerCase().trim()
                        def sourceUsername = params.SOURCE_USERNAME.trim()
                        def sourceToken = env.SOURCE_TOKEN
                        def destUsername = params.DEST_USERNAME.trim()
                        def destToken = env.DEST_TOKEN
                        def sourceWorkspace = params.SOURCE_WORKSPACE.trim()
                        def destOrganization = params.DEST_ORGANIZATION.trim()
                        def useSsh = params.USE_SSH

                        echo "DEBUG: Source platform is '${sourcePlatform}'"
                        echo "DEBUG: Destination platform is '${destPlatform}'"

                        // Fetch list of repositories
                        def repos
                        if (sourcePlatform == 'bitbucket') {
                            repos = sh(script: "curl -s -u '${sourceUsername}:${sourceToken}' 'https://api.bitbucket.org/2.0/repositories/${sourceWorkspace}' | jq -r '.values[].name'", returnStdout: true).trim()
                        } else {
                            repos = sh(script: "curl -s -u '${sourceUsername}:${sourceToken}' 'https://api.github.com/users/${sourceWorkspace}/repos' | jq -r '.[].name'", returnStdout: true).trim()
                        }

                        if (repos.isEmpty()) {
                            error "No repositories found or authentication failed."
                        }

                        // Sync each repository
                        repos.split('\n').each { repo ->
                            repo = repo.trim()
                            echo "Syncing repository '${repo}'..."

                            // Clean up the directory if it already exists
                            sh "rm -rf '${repo}.git' || true"

                            def sourceRepoUrl = createRepoUrl(sourcePlatform, sourceWorkspace, repo, useSsh, sourceUsername, sourceToken)
                            sh "git clone --bare '${sourceRepoUrl}' '${repo}.git'"

                            dir("${repo}.git") {
                                def destRepoUrl = createRepoUrl(destPlatform, destOrganization, repo, useSsh, destUsername, destToken)

                                def repoExists
                                if (destPlatform == 'bitbucket') {
                                    repoExists = sh(script: "curl -s -u '${destUsername}:${destToken}' 'https://api.bitbucket.org/2.0/repositories/${destOrganization}/${repo}' | jq -r '.name' || true", returnStdout: true).trim()
                                    if (repoExists == 'null') {
                                        echo "Creating repository '${repo}' on Bitbucket..."
                                        sh "curl -s -u '${destUsername}:${destToken}' 'https://api.bitbucket.org/2.0/repositories/${destOrganization}/${repo}' -X POST -H 'Content-Type: application/json' -d '{\"scm\": \"git\", \"is_private\": true}'"
                                    }
                                } else {
                                    repoExists = sh(script: "curl -s -u '${destUsername}:${destToken}' 'https://api.github.com/repos/${destOrganization}/${repo}' | jq -r '.name' || true", returnStdout: true).trim()
                                    if (repoExists == 'null') {
                                        echo "Creating repository '${repo}' on GitHub..."
                                        sh "curl -s -u '${destUsername}:${destToken}' 'https://api.github.com/user/repos' -d '{\"name\":\"${repo}\"}'"
                                    }
                                }

                                echo "Pushing changes to '${repo}' on ${destPlatform}..."
                                sh "git push --all '${destRepoUrl}'"
                                sh "git push --tags '${destRepoUrl}'"
                            }
                            sh "rm -rf '${repo}.git'"
                        }

                        echo "All repositories have been synced."
                    }
                }
            }
        }
    }
}
