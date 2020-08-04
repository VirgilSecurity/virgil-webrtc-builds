#!groovy


// --------------------------------------------------------------------------
//  Configuration properties.
// --------------------------------------------------------------------------
properties([
    parameters([
        booleanParam(name: 'CLEAN_BUILD', defaultValue: false,
            description: 'Remove all fetched toolchains.'),

        booleanParam(name: 'SKIP_BUILD_MACOS', defaultValue: false,
            description: 'Skip MacOS builds.'),

        stringParam(name: 'WEBRTC_VERSION', defaultValue: "4147",
            description: 'WebRTC version to build (check https://chromiumdash.appspot.com/releases)'),

        booleanParam(name: 'REBUILD_LINUX_DOCKER', defaultValue: false,
            description: 'Force to rebuild Docker container for Linux and Android builds.'),
    ])
])

// --------------------------------------------------------------------------
//  Helper functions
// --------------------------------------------------------------------------
def pathFromJobName(jobName) {
    return jobName.replace('/','-').replace('%2f', '-').replace('%2F', '-')
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
nodes['linux'] = build_linux('build-docker')

stage('Build') {
    parallel(nodes)
}

def build_macos(slave) {
    return { node(slave) {
        def jobPath = pathFromJobName(env.JOB_NAME)

        if (params.SKIP_BUILD_MACOS) {
            echo "MacOS builds are skipped."
            return
        }

        ws("workspace/${jobPath}") {
            dir('scripts') {
                deleteDir()
                unstash 'src'
                sh 'ls -l'
            }

            stage('Cleanup') {
                if (params.CLEAN_BUILD) {
                    deleteDir()
                } else {
                    echo "Cleanup was skipped."
                }
            }

            stage('Fetch tools') {
                dir('depot_tools') {
                    if (params.CLEAN_BUILD) {
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
            withEnv(["PATH+DEPOT_TOOLS=${rootDir}/depot_tools"]) {
                stage("Fetch sources") {
                    if (params.CLEAN_BUILD) {
                        sh "fetch --nohooks webrtc"
                    } else {
                        echo "Cached sources are used."
                    }
                }

                stage("Sync") {
                    if (params.CLEAN_BUILD) {
                        dir('src') {
                            sh "git checkout refs/remotes/branch-heads/${params.WEBRTC_VERSION}"
                            sh 'gclient sync'
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
                        // args += 'target_os = "mac"\n'
                        args += 'use_custom_libcxx = false\n'
                        args += 'target_cpu = "x64"\n'

                        dir('out/Release') {
                            args += 'is_debug = false\n'
                            writeFile(file: "args.gn", text: args)
                        }

                        sh """
                            gn gen 'out/Release'
                            ninja -C 'out/Release'
                        """

                        dir('out/Debug') {
                            args += 'is_debug = true\n'
                            writeFile(file: "args.gn", text: args)
                        }

                        sh """
                            gn gen 'out/Debug'
                            ninja -C 'out/Debug'
                        """
                    }
                }

                stage("Package") {
                    dir('src') {
                        dir('package') {
                            deleteDir()
                        }

                        fileOperations([
                            fileCopyOperation(
                                flattenFiles: false,
                                includes: '**/*.h',
                                targetLocation: "package/include/webrtc"
                            ),

                            fileCopyOperation(
                                flattenFiles: true,
                                renameFiles: true,
                                includes: 'out/Debug/obj/libwebrtc.a',
                                targetLocation: "package/lib",
                                sourceCaptureExpression: /(libwebrtc)\.a/,
                                targetNameExpression: '$1_d.a'
                            ),

                            fileCopyOperation(
                                flattenFiles: true,
                                includes: 'out/Release/obj/libwebrtc.a',
                                targetLocation: "package/lib"
                            ),

                            fileCopyOperation(
                                flattenFiles: true,
                                includes: 'out/Debug/obj/libwebrtc_d.a',
                                targetLocation: "package/lib"
                            )
                        ])

                        archiveArtifacts artifacts: 'package/**', fingerprint: true
                    }
                }
            }
        }
    }}
}

def build_linux(slave) {
    return { node(slave) {
        def jobPath = pathFromJobName(env.JOB_NAME)

        ws("workspace/${jobPath}") {
            dir('scripts') {
                deleteDir()
                unstash 'src'
                sh 'ls -l'
            }

            def buildContainerName = 'virgil-linux-webrtc:0.1.0'
            def buildContainerHash = sh(returnStdout: true, script: "docker images -q ${buildContainerName}").trim()
            def buildContainer = null
            if (!buildContainerHash || params.REBUILD_LINUX_DOCKER) {
                stage('Build Linux Docker image.') {
                    dir('scripts') {
                        buildContainer = docker.builds buildContainerName
                    }
                }
            } else {
                buildContainer = docker.image(buildContainerName)
            }

            dir('temp') {
                buildContainer.inside {
                    sh 'lsb_release > test.txt'
                }

                sh 'cat test.txt'
            }
        }
    }}
}
