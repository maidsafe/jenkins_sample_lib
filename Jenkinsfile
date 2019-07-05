properties([
    parameters([
        string(name: 'ARTIFACTS_BUCKET', defaultValue: 'safe-client-libs-jenkins'),
        string(name: 'DEPLOY_BUCKET', defaultValue: 'safe-cli')
    ])
])

stage('deploy') {
    node('docker') {
        checkout([
            $class: 'GitSCM',
            branches: scm.branches,
            doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
            extensions: scm.extensions + [[$class: 'CloneOption', noTags: false, reference: '', shallow: true]],
            submoduleCfg: [],
            userRemoteConfigs: scm.userRemoteConfigs])
        version = "0.0.29"
        retrieve_build_artifacts()
        if (version_change_commit()) {
            package_artifacts_for_deploy(true)
            create_tag(version)
            create_github_release(version)
        } else {
            package_artifacts_for_deploy(false)
            upload_deploy_artifacts()
        }
    }
}

def version_change_commit() {
    short_commit_hash = sh(
        returnStdout: true,
        script: "git log -n 1 --pretty=format:'%h'").trim()
    message = sh(
        returnStdout: true,
        script: "git log --format=%B -n 1 ${short_commit_hash}").trim()
    return message.startsWith("Version change")
}

def retrieve_build_artifacts() {
    command = "SAFE_CLI_BRANCH=113 "
    command += "SAFE_CLI_BUILD_NUMBER=10 "
    command += "make retrieve-all-build-artifacts"
    sh(command)
}

def package_artifacts_for_deploy(version_commit) {
    if (version_commit) {
        sh("make package-version-artifacts-for-deploy")
    } else {
        sh("make package-commit_hash-artifacts-for-deploy")
    }
}

def create_tag(version) {
    withCredentials([usernamePassword(
        credentialsId: "github_maidsafe_qa_user_credentials",
        usernameVariable: "GIT_USER",
        passwordVariable: "GIT_PASSWORD")]) {
        sh("git config --global user.name \$GIT_USER")
        sh("git config --global user.email qa@maidsafe.net")
        sh("git config credential.username \$GIT_USER")
        sh("git config credential.helper '!f() { echo password=\$GIT_PASSWORD; }; f'")
        sh("git tag -a ${version} -m 'Creating tag for ${version}'")
        sh("GIT_ASKPASS=true git push origin --tags")
    }
}

def create_github_release(version) {
    withCredentials([usernamePassword(
        credentialsId: "github_maidsafe_token_credentials",
        usernameVariable: "GITHUB_USER",
        passwordVariable: "GITHUB_TOKEN")]) {
        sh("make deploy-github-release")
    }
}

def upload_deploy_artifacts() {
    withAWS(credentials: 'aws_jenkins_deploy_artifacts_user', region: 'eu-west-2') {
        def artifacts = sh(returnStdout: true, script: 'ls -1 artifacts').trim().split("\\r?\\n")
        for (artifact in artifacts) {
            s3Upload(
                bucket: "${params.DEPLOY_BUCKET}",
                file: artifact,
                workingDir: "${env.WORKSPACE}/artifacts",
                acl: 'PublicRead')
        }
    }
}
