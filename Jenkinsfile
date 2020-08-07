#!groovy


// --------------------------------------------------------------------------
//  Configuration properties.
// --------------------------------------------------------------------------
properties([
    parameters([
        booleanParam(name: 'CLEAN_BUILD', defaultValue: false,
            description: 'Remove all fetched toolchains and build directories.'),

        booleanParam(name: 'CLEAN_WEBRTC_TOOLS', defaultValue: false,
            description: 'Remove fetch a new WebRTC Depot Tools.'),

        booleanParam(name: 'SKIP_BUILD_MACOS', defaultValue: false,
            description: 'Skip MacOS builds.'),

        booleanParam(name: 'SKIP_BUILD_LINUX', defaultValue: false,
            description: 'Skip Linux builds.'),

        booleanParam(name: 'SKIP_BUILD_ANDROID', defaultValue: false,
            description: 'Skip Android builds.'),

        booleanParam(name: 'REBUILD_LINUX_DOCKER', defaultValue: false,
            description: 'Force to rebuild Docker container for Linux and Android builds.'),

        stringParam(name: 'WEBRTC_VERSION', defaultValue: "4147",
            description: 'WebRTC version to build (check https://chromiumdash.appspot.com/releases)'),
    ])
])

// --------------------------------------------------------------------------
//  Helper functions
// --------------------------------------------------------------------------
def pathFromJobName(jobName) {
    return jobName.replace('/','-').replace('%2f', '-').replace('%2F', '-')
}

@NonCPS
def formatLeft(String str) {
    def res = ""

    str.eachLine { line, count ->
        res += line.trim() + '\n'
    }

    echo res

    return res
}

// --------------------------------------------------------------------------
//  Grab SCM
// --------------------------------------------------------------------------
node('master') {
    stage('Grab SCM') {
        env
        echo "PARAMETERS:"
        echo "CLEAN_BUILD = ${params.CLEAN_BUILD}"
        echo "WEBRTC_VERSION = ${params.WEBRTC_VERSION}"
        checkout scm
        stash includes: '**', name: 'src'
    }
}

// --------------------------------------------------------------------------
//  Build
// --------------------------------------------------------------------------
def nodes = [:]

nodes['macos'] = build_macos('build-os-x')
nodes['linux_android'] = build_linux_android('build-docker')


stage('Build') {
    parallel(nodes)
}


def fetchWebRtcTools() {
    stage('Fetch tools') {
        dir('depot_tools') {
            if (params.CLEAN_BUILD || params.CLEAN_WEBRTC_TOOLS || !fileExists('.git')) {
                deleteDir()
                checkout([
                    $class: 'GitSCM',
                    userRemoteConfigs: [[url: 'https://chromium.googlesource.com/chromium/tools/depot_tools.git']]
                ])
            } else {
                echo "Cached depot_tools are used."
            }
        }
    }

    return pwd() + '/depot_tools'
}

def inner_build_unix(webrtc, platform, archs) {
    dir('scripts') {
        deleteDir()
        unstash 'src'
    }

    dir("build/${platform}") {
        sh 'echo ${PATH}'

        def webRtcVersionFile = pwd() + '/WEBRTC_VERSION'

        stage('Cleanup') {
            if (params.CLEAN_BUILD) {
                deleteDir()
            } else {
                echo "Cleanup was skipped."
            }
        }

        stage("Fetch sources") {
            if (params.CLEAN_BUILD || !fileExists('src')) {
                sh "fetch --nohooks ${webrtc}"
            } else {
                echo "Cached sources are used."
            }
        }

        stage("Sync") {
            def isSyncRequired = !fileExists(webRtcVersionFile) || (readFile(file: webRtcVersionFile).trim() != params.WEBRTC_VERSION)
            if (params.CLEAN_BUILD || isSyncRequired) {
                dir('src') {
                    sh "git checkout refs/remotes/branch-heads/${params.WEBRTC_VERSION}"
                    sh 'gclient sync'
                    writeFile(file: webRtcVersionFile, text: params.WEBRTC_VERSION)
                }
            } else {
                echo "Sync sources were skipped."
            }
        }

        dir("out") {
            deleteDir()
        }

        stage("Compile") {
            dir('src') {
                def args = ""
                args += "target_os = \"${platform}\"\n"
                args += 'use_custom_libcxx = false\n'

                archs.each { arch ->
                    args += "target_cpu = \"${arch}\"\n"

                    dir("out/Release/${arch}") {
                        args += 'is_debug = false\n'
                        writeFile(file: "args.gn", text: args)
                    }

                    dir("out/Debug/${arch}") {
                        args += 'is_debug = true\n'
                        writeFile(file: "args.gn", text: args)
                    }

                    sh """
                        gn gen 'out/Release/${arch}'
                        ninja -C 'out/Release/${arch}'

                        gn gen 'out/Debug/${arch}'
                        ninja -C 'out/Debug/${arch}'
                    """
                }
            }
        }

        stage("Package") {
            dir('src') {
                dir('package/${platform}') {
                    deleteDir()
                }

                fileOperations([
                    fileCopyOperation(
                        flattenFiles: false,
                        includes: '**/*.h',
                        targetLocation: "package/${platform}/include/webrtc"
                    )
                ])

                archs.each { arch ->
                    fileOperations([fileCopyOperation(
                        flattenFiles: true,
                        renameFiles: true,
                        includes: "out/Debug/${arch}/obj/libwebrtc.a",
                        targetLocation: "package/${platform}/${arch}/lib",
                        sourceCaptureExpression: /(libwebrtc)\.a/,
                        targetNameExpression: '$1_d.a'
                    )])

                    fileOperations([fileCopyOperation(
                        flattenFiles: true,
                        includes: "out/Release/${arch}/obj/libwebrtc.a",
                        targetLocation: "package/${platform}/${arch}/lib"
                    )])

                    fileOperations([fileCopyOperation(
                        flattenFiles: true,
                        includes: "out/Debug/${arch}/obj/libwebrtc_d.a",
                        targetLocation: "package/${platform}/${arch}/lib"
                    )])
                }

                dir('package') {
                    archiveArtifacts artifacts: "${platform}/**", fingerprint: false
                }
            }
        }
    }
}

def build_macos(slave) {
    return { node(slave) {
        def jobPath = pathFromJobName(env.JOB_NAME)

        ws("workspace/${jobPath}") {
            if (params.SKIP_BUILD_MACOS) {
                echo "MacOS builds are skipped."
                return
            }

            def toolsPath = fetchWebRtcTools()

            stage('Build for MacOS') {
                withEnv(["PATH+WEBRTC_TOOLS=${toolsPath}"]) {
                    inner_build_unix("webrtc", "mac", ["x64"])
                }
            }
        }
    }}
}

def build_linux_android(slave) {
    return { node(slave) {
        def jobPath = pathFromJobName(env.JOB_NAME)

        ws("workspace/${jobPath}") {
            def toolsPath = fetchWebRtcTools()

            def buildContainerName = 'virgil-linux-webrtc:0.1.0'

            stage('Build Linux Docker image.') {
                dir('scripts') {
                    deleteDir()
                    unstash 'src'

                    dir('linux_android') {
                        def buildContainerHash =
                                sh(returnStdout: true, script: "docker images -q ${buildContainerName}").trim()

                        if (!buildContainerHash || params.REBUILD_LINUX_DOCKER) {
                            docker.build(buildContainerName)
                        }
                    }
                }
            }

            def buildContainer = null
            dir('docker') {
                writeFile(file: 'Dockerfile', text: formatLeft("""
                    FROM ${buildContainerName}
                    ENV PATH=\"${toolsPath}:\${PATH}\"
                """
                ))

                buildContainer = docker.build('virgil-linux-webrtc-tmp')
            }

            buildContainer.inside {
                stage('Build for Linux') {
                    if (!params.SKIP_BUILD_LINUX) {
                        inner_build_unix("webrtc", "linux", ["x64"])
                    } else {
                        echo "Linux builds are skipped."
                    }
                }

                stage('Build for Android') {
                    if (!params.SKIP_BUILD_ANDROID) {
                        inner_build_unix("webrtc_android", "android", ["arm", "arm64", "x86", "x64"])
                    } else {
                        echo "Android builds are skipped."
                    }
                }
            }
        }
    }}
}
