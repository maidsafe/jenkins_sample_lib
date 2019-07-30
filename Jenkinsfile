properties([
    parameters([
        string(name: 'ARTIFACTS_BUCKET', defaultValue: 'safe-jenkins-build-artifacts')
    ])
])

stage('build & test') {
    parallel linux: {
        node('linux') {
            checkout(scm)
            sh("make test")
            packageBuildArtifacts('linux')
            uploadBuildArtifacts()
        }
    },
    windows: {
        node('windows') {
            checkout(scm)
            sh("make test")
            packageBuildArtifacts('linux')
            uploadBuildArtifacts()
        }
    },
    macos: {
        node('osx') {
            checkout(scm)
            sh("make test")
            packageBuildArtifacts('macos')
            uploadBuildArtifacts()
        }
    }
}

stage('deploy') {
    node('docker') {
        checkout(scm)
        retrieveBuildArtifacts()
        withCredentials([string(
            credentialsId: 'crates_io_token', variable: 'CRATES_IO_TOKEN')]) {
            sh("make publish")
        }
    }
}

def packageBuildArtifacts(os) {
    branch = env.CHANGE_ID?.trim() ?: env.BRANCH_NAME
    withEnv(["JENKINS_SAMPLE_BRANCH=${branch}",
             "JENKINS_SAMPLE_BUILD_NUMBER=${env.BUILD_NUMBER}",
             "JENKINS_SAMPLE_BUILD_OS=${os}"]) {
        sh("make package-build-artifacts")
    }
}

def uploadBuildArtifacts() {
    withAWS(credentials: 'aws_jenkins_build_artifacts_user', region: 'eu-west-2') {
        def artifacts = sh(returnStdout: true, script: 'ls -1 artifacts').trim().split("\\r?\\n")
        for (artifact in artifacts) {
            s3Upload(
                bucket: "${params.ARTIFACTS_BUCKET}",
                file: artifact,
                workingDir: "${env.WORKSPACE}/artifacts",
                acl: 'PublicRead')
        }
    }
}

def retrieveBuildArtifacts() {
    branch = env.CHANGE_ID?.trim() ?: env.BRANCH_NAME
    withEnv(["JENKINS_SAMPLE_BRANCH=${branch}",
             "JENKINS_SAMPLE_BUILD_NUMBER=${env.BUILD_NUMBER}"]) {
        sh("make retrieve-all-build-artifacts")
    }
}
