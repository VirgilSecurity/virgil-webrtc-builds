#!groovy


// --------------------------------------------------------------------------
//  Configuration properties.
// --------------------------------------------------------------------------
properties([
    parameters([
        booleanParam(name: 'CLEAN_BUILD', defaultValue: false,
            description: 'Remove all fetched toolchains and build directories.'),

        booleanParam(name: 'SKIP_BUILD_MACOS', defaultValue: false,
            description: 'Skip MacOS builds.'),

        booleanParam(name: 'SKIP_BUILD_LINUX', defaultValue: false,
            description: 'Skip Linux builds.'),

        booleanParam(name: 'SKIP_ANDROID_LINUX', defaultValue: false,
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

def envSh(env, command) {
    sh """
        export ${env}
        ${command}
    """
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


def inner_build_unix(webrtc, platform, archs) {
    dir('scripts') {
        deleteDir()
        unstash 'src'
    }

    dir("build/${platform}") {
        stage('Cleanup') {
            if (params.CLEAN_BUILD) {
                deleteDir()
            } else {
                echo "Cleanup was skipped."
            }
        }

        stage('Fetch tools') {
            dir('depot_tools') {
                if (params.CLEAN_BUILD || !fileExists('depot_tools/')) {
                    checkout([
                        $class: 'GitSCM',
                        userRemoteConfigs: [[url: 'https://chromium.googlesource.com/chromium/tools/depot_tools.git']]
                    ])
                } else {
                    echo "Cached depot_tools are used."
                }
            }
        }

        def rootDir = pwd()

        // withEnv() can not be used due to the bug: https://issues.jenkins-ci.org/browse/JENKINS-49076
        def envPath = "PATH=${rootDir}/depot_tools:$PATH"

        env.PATH = "${rootDir}/depot_tools:${env.PATH}"

        sh 'echo ----------'
        sh 'echo ${PATH}'

        envSh(envPath, "echo $PATH")

        stage("Fetch sources") {
            if (params.CLEAN_BUILD || !fileExists('src')) {
                envSh(envPath, "fetch --nohooks ${webrtc}")
            } else {
                echo "Cached sources are used."
            }
        }

        stage("Sync") {
            if (params.CLEAN_BUILD) {
                dir('src') {
                    envSh(envPath, "git checkout refs/remotes/branch-heads/${params.WEBRTC_VERSION}")
                    envSh(envPath, 'gclient sync')
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

                    envSh(envPath, """
                        gn gen 'out/Release/${arch}'
                        ninja -C 'out/Release/${arch}'

                        gn gen 'out/Debug/${arch}'
                        ninja -C 'out/Debug/${arch}'
                    """)
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
                    archiveArtifacts artifacts: "${platform}/**", fingerprint: true
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

            stage('Build for MacOS') {
                inner_build_unix("webrtc", "mac", ["x64"])
            }
        }
    }}
}

def build_linux_android(slave) {
    return { node(slave) {
        def jobPath = pathFromJobName(env.JOB_NAME)

        ws("workspace/${jobPath}") {
            def buildContainerName = 'virgil-linux-webrtc:0.1.0'

            dir('scripts') {
                deleteDir()
                unstash 'src'

                dir('linux_android') {
                    def buildContainerHash = sh(returnStdout: true, script: "docker images -q ${buildContainerName}").trim()

                    if (!buildContainerHash || params.REBUILD_LINUX_DOCKER) {
                        stage('Build Linux Docker image.') {
                            docker.build(buildContainerName)
                        }
                    }
                }
            }

            def buildContainer = docker.image(buildContainerName)

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
