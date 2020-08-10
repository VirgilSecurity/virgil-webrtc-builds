#!groovy


// --------------------------------------------------------------------------
//  Configuration properties.
// --------------------------------------------------------------------------
properties([
    parameters([
        booleanParam(name: 'CLEAN_WEBRTC_TOOLS', defaultValue: false,
            description: 'Fetch a new WebRTC Depot Tools.'),

        booleanParam(name: 'CLEAN_WEBRTC_SOURCES', defaultValue: false,
            description: 'Fetch a new WebRTC sources.'),

        booleanParam(name: 'CLEAN_BUILD', defaultValue: false,
            description: 'Remove build directories.'),

        booleanParam(name: 'SKIP_BUILD_MACOS', defaultValue: false,
            description: 'Skip MacOS builds.'),

        booleanParam(name: 'SKIP_BUILD_IOS', defaultValue: false,
            description: 'Skip iOS builds.'),

        booleanParam(name: 'SKIP_BUILD_LINUX', defaultValue: false,
            description: 'Skip Linux builds.'),

        booleanParam(name: 'SKIP_BUILD_ANDROID', defaultValue: false,
            description: 'Skip Android builds.'),

        stringParam(name: 'WEBRTC_VERSION', defaultValue: "4147",
            description: 'WebRTC version to build (check https://chromiumdash.appspot.com/releases)'),
    ])
])

// --------------------------------------------------------------------------
//  Helper functions
// --------------------------------------------------------------------------
def path_from_job_name(jobName) {
    return jobName.replace('/','-').replace('%2f', '-').replace('%2F', '-')
}

@NonCPS
def format_left(String str) {
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
        echo "=== PARAMETERS: ==="
        echo "CLEAN_WEBRTC_TOOLS = ${params.CLEAN_WEBRTC_TOOLS}"
        echo "CLEAN_WEBRTC_SOURCES = ${params.CLEAN_WEBRTC_SOURCES}"
        echo "CLEAN_BUILD = ${params.CLEAN_BUILD}"
        echo "SKIP_BUILD_MACOS = ${params.SKIP_BUILD_MACOS}"
        echo "SKIP_BUILD_IOS = ${params.SKIP_BUILD_IOS}"
        echo "SKIP_BUILD_LINUX = ${params.SKIP_BUILD_LINUX}"
        echo "SKIP_BUILD_ANDROID = ${params.SKIP_BUILD_ANDROID}"
        echo "WEBRTC_VERSION = ${params.WEBRTC_VERSION}"
        echo "==="

        checkout scm
        stash includes: '**', name: 'src'
    }
}

// --------------------------------------------------------------------------
//  Build
// --------------------------------------------------------------------------
def nodes = [:]

nodes['macos'] = build_macos('build-os-x')
nodes['linux_android'] = build_linux_android('build-ubuntu20')

stage('Build') {
    parallel(nodes)
}

def fetch_webrtc_tools() {
    stage('Fetch tools') {
        dir('depot_tools') {
            if (params.CLEAN_WEBRTC_TOOLS || !fileExists('.git')) {
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

def inner_build_unix(webrtc, platform, archs, options = []) {
    dir('scripts') {
        deleteDir()
        unstash 'src'
    }

    dir("build/${platform}") {
        sh 'echo ${PATH}'

        def webRtcVersionFile = pwd() + '/WEBRTC_VERSION'

        stage("Fetch sources") {
            if (params.CLEAN_WEBRTC_SOURCES) {
                dir('src') {
                    deleteDir()
                }
            }

            if (params.CLEAN_WEBRTC_SOURCES || !fileExists('src')) {
                sh "fetch --nohooks ${webrtc}"
            } else {
                echo "Cached sources are used."
            }
        }

        stage("Sync") {
            def isSyncRequired = !fileExists(webRtcVersionFile) ||
                                 (readFile(file: webRtcVersionFile).trim() != params.WEBRTC_VERSION)

            if (params.CLEAN_WEBRTC_SOURCES || isSyncRequired) {
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

                options.each { option ->
                    args += "${option}\n"
                }

                archs.each { arch ->
                    args += "target_cpu = \"${arch}\"\n"

                    dir("out/Release/${arch}") {
                        if (params.CLEAN_BUILD) {
                            deleteDir()
                        }

                        args += 'is_debug = false\n'
                        writeFile(file: "args.gn", text: args)
                    }

                    dir("out/Debug/${arch}") {
                        if (params.CLEAN_BUILD) {
                            deleteDir()
                        }

                        args += 'is_debug = true\n'
                        writeFile(file: "args.gn", text: args)
                    }

                    sh """
                        gn gen 'out/Release/${arch}'
                        gn gen 'out/Debug/${arch}'

                        ninja -C 'out/Release/${arch}' webrtc
                        ninja -C 'out/Debug/${arch}' webrtc
                    """
                }
            }
        }
    }
}

def inner_pack_unix(platform) {
    stage("Pack/Archive") {
        dir("build/${platform}/src") {
            dir("package/${platform}") {
                deleteDir()
            }

            //
            //  Copy includes.
            //
            fileOperations([
                fileCopyOperation(
                    flattenFiles: false,
                    includes: '**/*.h',
                    targetLocation: "package/${platform}/include/webrtc"
                )
            ])

            //
            //  Pack Release libraries.
            //
            if (fileExists('out/Release/x86/obj/libwebrtc.a')) {
                sh "cp out/Release/x86/obj/libwebrtc.a package/${platform}/lib/x86/libwebrtc.a"
            }

            if (fileExists('out/Release/x64/obj/libwebrtc.a')) {
                sh "cp out/Release/x64/obj/libwebrtc.a package/${platform}/lib/x86_64/libwebrtc.a"
            }

            //
            //  Pack Debug libraries.
            //
            if (fileExists('out/Debug/x86/obj/libwebrtc.a')) {
                sh "cp out/Debug/x86/obj/libwebrtc.a package/${platform}/lib/x86/libwebrtc_d.a"
            }

            if (fileExists('out/Debug/x64/obj/libwebrtc.a')) {
                sh "cp out/Debug/x64/obj/libwebrtc.a package/${platform}/lib/x86_64/libwebrtc_d.a"
            }

            //
            //  Acrhive.
            //
            dir('package') {
                archiveArtifacts artifacts: "${platform}/**", fingerprint: false
            }
        }
    }
}

def inner_pack_macos_ios(platform) {
    stage("Pack/Archive") {
        dir("build/${platform}/src") {
            dir("package/${platform}") {
                deleteDir()
            }

            //
            //  Copy includes.
            //
            fileOperations([
                fileCopyOperation(
                    flattenFiles: false,
                    includes: '**/*.h',
                    targetLocation: "package/${platform}/include/webrtc"
                )
            ])

            //
            //  Pack Release libraries.
            //

            sh "mkdir -p package/${platform}/lib"

            def releaseLibs = ""
            if (fileExists('out/Release/x86/obj/libwebrtc.a')) {
                releaseLibs += " out/Release/x86/obj/libwebrtc.a"
            }

            if (fileExists('out/Release/x64/obj/libwebrtc.a')) {
                releaseLibs += " out/Release/x64/obj/libwebrtc.a"
            }

            if (fileExists('out/Release/arm/obj/libwebrtc.a')) {
                releaseLibs += " out/Release/arm/obj/libwebrtc.a"
            }

            if (fileExists('out/Release/arm64/obj/libwebrtc.a')) {
                releaseLibs += " out/Release/arm64/obj/libwebrtc.a"
            }

            sh "xcrun lipo -create ${releaseLibs} -output package/${platform}/lib/libwebrtc.a"

            //
            //  Pack Debug libraries.
            //
            def debugLibs = ""
            if (fileExists('out/Debug/x86/obj/libwebrtc.a')) {
                debugLibs += " out/Debug/x86/obj/libwebrtc.a"
            }

            if (fileExists('out/Debug/x64/obj/libwebrtc.a')) {
                debugLibs += " out/Debug/x64/obj/libwebrtc.a"
            }

            if (fileExists('out/Debug/arm/obj/libwebrtc.a')) {
                debugLibs += " out/Debug/arm/obj/libwebrtc.a"
            }

            if (fileExists('out/Debug/arm64/obj/libwebrtc.a')) {
                debugLibs += " out/Debug/arm64/obj/libwebrtc.a"
            }

            sh "xcrun lipo -create ${debugLibs} -output package/${platform}/lib/libwebrtc_d.a"

            //
            //  Acrhive.
            //
            dir('package') {
                archiveArtifacts artifacts: "${platform}/**", fingerprint: false
            }
        }
    }
}

def inner_pack_android(platform) {
    stage("Pack/Archive") {
        dir("build/${platform}/src") {
            dir("package/${platform}") {
                deleteDir()
            }

            //
            //  Copy includes.
            //
            fileOperations([
                fileCopyOperation(
                    flattenFiles: false,
                    includes: '**/*.h',
                    targetLocation: "package/${platform}/include/webrtc"
                )
            ])

            //
            //  Pack Release libraries.
            //
            if (fileExists('out/Release/x86/obj/libwebrtc.a')) {
                sh """
                    mkdir -p package/${platform}/lib/x86
                    cp out/Release/x86/obj/libwebrtc.a package/${platform}/lib/x86/libwebrtc.a
                """
            }

            if (fileExists('out/Release/x64/obj/libwebrtc.a')) {
                sh """
                    mkdir -p package/${platform}/lib/x86_64
                    cp out/Release/x64/obj/libwebrtc.a package/${platform}/lib/x86_64/libwebrtc.a
                """
            }

            if (fileExists('out/Release/arm/obj/libwebrtc.a')) {
                sh """
                    mkdir -p package/${platform}/lib/armeabi-v7a
                    cp out/Release/arm/obj/libwebrtc.a package/${platform}/lib/armeabi-v7a/libwebrtc.a
                """
            }

            if (fileExists('out/Release/arm64/obj/libwebrtc.a')) {
                sh """
                    mkdir -p package/${platform}/lib/arm64-v8a
                    cp out/Release/arm64/obj/libwebrtc.a package/${platform}/lib/arm64-v8a/libwebrtc.a
                """
            }

            //
            //  Pack Debug libraries.
            //
            if (fileExists('out/Debug/x86/obj/libwebrtc.a')) {
                sh """
                    mkdir -p package/${platform}/lib/x86
                    cp out/Debug/x86/obj/libwebrtc.a package/${platform}/lib/x86/libwebrtc_d.a
                """
            }

            if (fileExists('out/Debug/x64/obj/libwebrtc.a')) {
                sh """
                    mkdir -p package/${platform}/lib/x86_64
                    cp out/Debug/x64/obj/libwebrtc.a package/${platform}/lib/x86_64/libwebrtc_d.a
                """
            }

            if (fileExists('out/Debug/arm/obj/libwebrtc.a')) {
                sh """
                    mkdir -p package/${platform}/lib/armeabi-v7a
                    cp out/Debug/arm/obj/libwebrtc.a package/${platform}/lib/armeabi-v7a/libwebrtc_d.a
                """
            }

            if (fileExists('out/Debug/arm64/obj/libwebrtc.a')) {
                sh """
                    mkdir -p package/${platform}/lib/arm64-v8a
                    cp out/Debug/arm64/obj/libwebrtc.a package/${platform}/lib/arm64-v8a/libwebrtc_d.a
                """
            }

            //
            //  Acrhive.
            //
            dir('package') {
                archiveArtifacts artifacts: "${platform}/**", fingerprint: false
            }
        }
    }
}

def build_macos(slave) {
    return { node(slave) {
        def jobPath = path_from_job_name(env.JOB_NAME)

        ws("workspace/${jobPath}") {
            def toolsPath = fetch_webrtc_tools()

            withEnv(["PATH+WEBRTC_TOOLS=${toolsPath}"]) {
                stage('Build for MacOS') {
                    if (!params.SKIP_BUILD_MACOS) {
                        inner_build_unix("webrtc", "mac", ["x64"])
                        inner_pack_macos_ios("mac")
                    } else {
                        echo "MacOS builds are skipped."
                    }
                }

                stage('Build for iOS') {
                    if (!params.SKIP_BUILD_IOS) {
                        inner_build_unix(
                                "webrtc_ios", "ios", ["arm", "x86", "arm64", "x64"],
                                ["enable_ios_bitcode=true", "use_xcode_clang=true", "ios_enable_code_signing=false"]
                        )
                        inner_pack_macos_ios("ios")
                    } else {
                        echo "Android builds are skipped."
                    }
                }
            }
        }
    }}
}

def build_linux_android(slave) {
    return { node(slave) {
        def jobPath = path_from_job_name(env.JOB_NAME)

        ws("workspace/${jobPath}") {
            def toolsPath = fetch_webrtc_tools()

            withEnv(["PATH+WEBRTC_TOOLS=${toolsPath}"]) {
                stage('Build for Linux') {
                    if (!params.SKIP_BUILD_LINUX) {
                        inner_build_unix("webrtc", "linux", ["x64"])
                        inner_pack_unix("linux")
                    } else {
                        echo "Linux builds are skipped."
                    }
                }

                stage('Build for Android') {
                    if (!params.SKIP_BUILD_ANDROID) {
                        inner_build_unix("webrtc_android", "android", ["arm", "x86", "arm64", "x64"])
                        inner_pack_android("android")
                    } else {
                        echo "Android builds are skipped."
                    }
                }
            }
        }
    }}
}
