pipeline {
    agent none
    options {
        // TODO: set max. no. of concurrent builds to 2
        timeout(time: 12, unit: "HOURS")
        timestamps()
    }
    parameters {
        choice(name: "CHANNEL", choices: ["nightly", "dev", "beta", "release"], description: "")
        choice(name: "BUILD_TYPE", choices: ["Release", "Debug"], description: "")
        booleanParam(name: "WIPE_WORKSPACE", defaultValue: false, description: "")
        booleanParam(name: "SKIP_INIT", defaultValue: false, description: "")
        booleanParam(name: "DISABLE_SCCACHE", defaultValue: false, description: "")
        booleanParam(name: "SKIP_SIGNING", defaultValue: true, description: "")
        booleanParam(name: "DEBUG", defaultValue: false, description: "")
    }
    environment {
        REFERRAL_API_KEY = credentials("REFERRAL_API_KEY")
        BRAVE_GOOGLE_API_KEY = credentials("npm_config_brave_google_api_key")
        BRAVE_ARTIFACTS_BUCKET = credentials("brave-jenkins-artifacts-s3-bucket")
        BRAVE_S3_BUCKET = credentials("brave-binaries-s3-bucket")
        SLACK_USERNAME_MAP = credentials("github-to-slack-username-map")
    }
    stages {
        stage("env") {
            steps {
                script {
                    CHANNEL = params.CHANNEL
                    CHANNEL_CAPITALIZED = CHANNEL.equals("release") ? "" : CHANNEL.capitalize()
                    CHANNEL_CAPITALIZED_BACKSLASHED_SPACED = CHANNEL.equals("release") ? "" : "\\ " + CHANNEL.capitalize()
                    BUILD_TYPE = params.BUILD_TYPE
                    WIPE_WORKSPACE = params.WIPE_WORKSPACE
                    SKIP_INIT = params.SKIP_INIT
                    DISABLE_SCCACHE = params.DISABLE_SCCACHE
                    SKIP_SIGNING = params.SKIP_SIGNING ? "--skip_signing" : ""
                    DEBUG = params.DEBUG
                    RELEASE_TYPE = env.JOB_NAME.equals("brave-browser-build") ? "release" : "ci"
                    OUT_DIR = "src/out/" + BUILD_TYPE
                    BUILD_TAG_SLASHED = env.JOB_NAME + "/" + env.BUILD_NUMBER
                    LINT_BRANCH = "TEMP_LINT_BRANCH_" + env.BUILD_NUMBER
                    BRAVE_GITHUB_TOKEN = "brave-browser-releases-github"
                    GITHUB_API = "https://api.github.com/repos/brave"
                    GITHUB_CREDENTIAL_ID = "brave-builds-github-token-for-pr-builder"
                    RUST_LOG = "sccache=warn"
                    SKIP = false
                    BRANCH = env.BRANCH_NAME
                    TARGET_BRANCH = "master"
                    if (env.CHANGE_BRANCH) {
                        BRANCH = env.CHANGE_BRANCH
                        TARGET_BRANCH = env.CHANGE_TARGET
                        def prNumber = readJSON(text: httpRequest(url: GITHUB_API + "/brave-browser/pulls?head=brave:" + BRANCH, authentication: GITHUB_CREDENTIAL_ID, quiet: !DEBUG).content)[0].number
                        def prDetails = readJSON(text: httpRequest(url: GITHUB_API + "/brave-browser/pulls/" + prNumber, authentication: GITHUB_CREDENTIAL_ID, quiet: !DEBUG).content)
                        SKIP = prDetails.mergeable_state.equals("draft") or prDetails.labels.count { label -> label.name.equals("CI/Skip") }.equals(1)
                        env.SLACK_USERNAME = readJSON(text: SLACK_USERNAME_MAP)[env.CHANGE_AUTHOR]
                        if (env.SLACK_USERNAME) {
                            slackSend(color: null, channel: env.SLACK_USERNAME, message: "STARTED " + BUILD_TAG_SLASHED + " (<${BUILD_URL}/flowGraphTable/?auto_refresh=true|Open>)")
                        }
                    }
                    BRANCH_EXISTS_IN_BC = httpRequest(url: GITHUB_API + "/brave-core/branches/" + BRANCH, validResponseCodes: "100:499", authentication: GITHUB_CREDENTIAL_ID, quiet: !DEBUG).status.equals(200)
                    if (BRANCH_EXISTS_IN_BC) {
                        def bcPrDetails = readJSON(text: httpRequest(url: GITHUB_API + "/brave-core/pulls?head=brave:" + BRANCH, authentication: GITHUB_CREDENTIAL_ID, quiet: !DEBUG).content)[0]
                        if (bcPrDetails) {
                            env.BC_PR_NUMBER = bcPrDetails.number
                        }
                    }
                }
            }
        }
        stage("abort") {
            steps {
                script {
                    if (SKIP) {
                        echo "Aborting build as PR is in draft or has \"CI/Skip\" label"
                        stopCurrentBuild()
                    }
                    else if (BRANCH_EXISTS_IN_BC) {
                        if (isStartedManually()) {
                            if (env.BC_PR_NUMBER) {
                                echo "Aborting build as PR exists in brave-core and build has not been started from there"
                                echo "Use " + env.JENKINS_URL + "view/ci/job/brave-core-build-pr/view/change-requests/job/PR-" + env.BC_PR_NUMBER + " to trigger"
                            }
                            else {
                                echo "Aborting build as there's a matching branch in brave-core, please create a PR there first"
                                echo "Use https://github.com/brave/brave-core/compare/" + TARGET_BRANCH + "..." + BRANCH + " to create PR"
                            }
                            SKIP = true
                            stopCurrentBuild()
                        }
                    }
                    def bb_package_json = readJSON(text: httpRequest(url: "https://raw.githubusercontent.com/brave/brave-browser/" + BRANCH + "/package.json", quiet: !DEBUG).content)
                    def bb_version = bb_package_json.version
                    def bc_branch = bb_package_json.config.projects["brave-core"].branch
                    if (BRANCH_EXISTS_IN_BC) {
                        bc_branch = BRANCH
                    }
                    def bc_version = readJSON(text: httpRequest(url: "https://raw.githubusercontent.com/brave/brave-core/" + bc_branch + "/package.json", quiet: !DEBUG).content).version
                    if (bb_version != bc_version) {
                        echo "Version mismatch between brave-browser (" + BRANCH + "/" + bb_version + ") and brave-core (" + bc_branch + "/" + bc_version + ") in package.json"
                        SKIP = true
                        stopCurrentBuild()
                    }
                    if (!SKIP) {
                        for (build in getBuilds()) {
                            if (build.isBuilding() && build.getNumber() < env.BUILD_NUMBER.toInteger()) {
                                echo "Aborting older running build " + build
                                build.doStop()
                                // build.finish(hudson.model.Result.ABORTED, new java.io.IOException("Aborting build"))
                            }
                        }
                        sleep(time: 1, unit: "MINUTES")
                    }
                }
            }
        }
        stage("build-all") {
            when {
                beforeAgent true
                expression { !SKIP }
            }
            parallel {
                stage("android") {
                    agent { label "mac-${RELEASE_TYPE}" }
                    environment {
                        GIT_CACHE_PATH = "${HOME}/cache"
                        // SCCACHE_BUCKET = credentials("brave-browser-sccache-android-s3-bucket")
                        SCCACHE_ERROR_LOG  = "${WORKSPACE}/sccache.log"
                    }
                    stages {
                        stage("checkout") {
                            when {
                                anyOf {
                                    expression { WIPE_WORKSPACE }
                                    expression { return !fileExists("package.json") }
                                }
                            }
                            steps {
                                checkout([$class: "GitSCM", branches: [[name: BRANCH]], extensions: [[$class: "WipeWorkspace"]], userRemoteConfigs: [[url: "https://github.com/brave/brave-browser.git"]]])
                            }
                        }
                        stage("pin") {
                            when {
                                expression { BRANCH_EXISTS_IN_BC }
                            }
                            steps {
                                echo "Pinning brave-core to use branch ${BRANCH}"
                                sh """
                                    set -e
                                    jq 'del(.config.projects["brave-core"].branch) | .config.projects["brave-core"].branch="${BRANCH}"' package.json > package.json.new
                                    mv package.json.new package.json
                                """
                            }
                        }
                        stage("install") {
                            steps {
                                sh "npm install --no-optional"
                                sh "rm -rf ${GIT_CACHE_PATH}/*.lock"
                            }
                        }
                        stage("init") {
                            when {
                                expression { return !fileExists("src/brave/package.json") || !SKIP_INIT }
                            }
                            steps {
                                sh """
                                    set -e
                                    rm -rf src/brave
                                    npm run init -- --target_os=ios
                                """
                            }
                        }
                        stage("lint") {
                            steps {
                                script {
                                    try {
                                        sh """
                                            set -e
                                            git -C src/brave config user.name brave-builds
                                            git -C src/brave config user.email devops@brave.com
                                            git -C src/brave checkout -b ${LINT_BRANCH}
                                            npm run lint -- --base=origin/${TARGET_BRANCH}
                                            git -C src/brave checkout -q -
                                            git -C src/brave branch -D ${LINT_BRANCH}
                                        """
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                }
                            }
                        }
                        stage("audit-deps") {
                            steps {
                                timeout(time: 1, unit: "MINUTES") {
                                    script {
                                        try {
                                            sh "npm run audit_deps"
                                        }
                                        catch (ex) {
                                            currentBuild.result = "UNSTABLE"
                                        }
                                    }
                                }
                            }
                        }
                        // stage("sccache") {
                        //     when {
                        //         allOf {
                        //             expression { !DISABLE_SCCACHE }
                        //             expression { RELEASE_TYPE.equals("ci") }
                        //         }
                        //     }
                        //     steps {
                        //         echo "Enabling sccache"
                        //         sh "npm config --userconfig=.npmrc set sccache sccache"
                        //     }
                        // }
                        stage("build") {
                            steps {
                                sh """
                                    set -e
                                    npm config --userconfig=.npmrc set brave_referrals_api_key ${REFERRAL_API_KEY}
                                    npm config --userconfig=.npmrc set brave_google_api_endpoint https://location.services.mozilla.com/v1/geolocate?key=
                                    npm config --userconfig=.npmrc set brave_google_api_key ${BRAVE_GOOGLE_API_KEY}
                                    npm config --userconfig=.npmrc set google_api_endpoint safebrowsing.brave.com
                                    npm config --userconfig=.npmrc set google_api_key dummytoken
                                    npm run build -- ${BUILD_TYPE} --channel=${CHANNEL} --official_build=true --target_os=ios
                                """
                            }
                        }
                        stage("archive") {
                            steps {
                                s3Upload(acl: "Private", bucket: BRAVE_ARTIFACTS_BUCKET, includePathPattern: "apks/*.apk", path: BUILD_TAG_SLASHED, workingDir: OUT_DIR)
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                if (env.SLACK_USERNAME) {
                    def slackColorMap = ["SUCCESS": "good", "FAILURE": "danger", "UNSTABLE": "warning", "ABORTED": null]
                    slackSend(color: slackColorMap[currentBuild.currentResult], channel: env.SLACK_USERNAME, message: currentBuild.currentResult + " " + BUILD_TAG_SLASHED + " (<${BUILD_URL}/flowGraphTable/?auto_refresh=true|Open>)")
                }
            }
        }
    }
}

@NonCPS
def stopCurrentBuild() {
    Jenkins.instance.getItemByFullName(env.JOB_NAME).getLastBuild().doStop()
}

@NonCPS
def isStartedManually() {
    return Jenkins.instance.getItemByFullName(env.JOB_NAME).getLastBuild().getCause(hudson.model.Cause$UpstreamCause) == null
}

@NonCPS
def getBuilds() {
    return Jenkins.instance.getItemByFullName(env.JOB_NAME).builds
}
