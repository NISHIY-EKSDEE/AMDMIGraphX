import org.jenkinsci.plugins.pipeline.modeldefinition.Utils

def DOCKER_IMAGE = 'vastrakhamd/ci-test'

def getgputargets() {
    targets="gfx906;gfx908;gfx90a;gfx1030;gfx1100;gfx1101;gfx1102"
    return targets
}

// def rocmtestnode(variant, name, body, args, pre) {
def rocmtestnode(Map conf) {
    def variant = conf.get("variant")
    def name = conf.get("node")
    def body = conf.get("body")
    def docker_args = conf.get("docker_args", "")
    def docker_build_args = conf.get("docker_build_args", "")
    def pre = conf.get("pre", {})
    def ccache = "/var/jenkins/.cache/ccache"
    def image = 'migraphxlib'
    env.CCACHE_COMPRESSLEVEL = 7
    env.CCACHE_DIR = ccache
    def cmake_build = { bconf ->
        def compiler = bconf.get("compiler", "/opt/rocm/llvm/bin/clang++")
        def flags = bconf.get("flags", "")
        def gpu_debug = bconf.get("gpu_debug", "0")
        def cmd = """
            ulimit -c unlimited
            echo "leak:dnnl::impl::malloc" > suppressions.txt
            export LSAN_OPTIONS="suppressions=\$(pwd)/suppressions.txt"
            export MIGRAPHX_GPU_DEBUG=${gpu_debug}
            export CXX=${compiler}
            export CXXFLAGS='-Werror'
            env
            rm -rf build
            mkdir build
            cd build
            cmake -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache -DBUILD_DEV=On -DCMAKE_EXECUTE_PROCESS_COMMAND_ECHO=STDOUT ${flags} ..
            git diff
            git diff-index --quiet HEAD || (echo "Git repo is not clean after running cmake." && exit 1)
            make -j\$(nproc) generate VERBOSE=1
            git diff
            git diff-index --quiet HEAD || (echo "Generated files are different. Please run make generate and commit the changes." && exit 1)
            make -j\$(nproc) all doc package check VERBOSE=1
            md5sum ./*.deb
        """
        echo cmd
        sh cmd
        // Only archive from master or develop
        if (env.BRANCH_NAME == "develop" || env.BRANCH_NAME == "master") {
            archiveArtifacts artifacts: "build/*.deb", allowEmptyArchive: true, fingerprint: true
        }
    }
    node(name) {
        withEnv(['HSA_ENABLE_SDMA=0']) {
            stage("checkout ${variant}") {
                checkout scm
            }
            gitStatusWrapper(credentialsId: "github-creds", gitHubContext: "Jenkins - ${variant}", account: 'NISHIY-EKSDEE', repo: 'AMDMIGraphX') {
                docker.withRegistry('https://registry.hub.docker.com', 'DOCKER_HUB_CREDS') {
                    pre()
                    try {
                        sh "docker pull ${DOCKER_IMAGE}:${env.IMAGE_TAG}"
                    } catch(Exception ex) {
                        println("Failed to pull")
                    }
                    withDockerContainer(image: "${DOCKER_IMAGE}:${env.IMAGE_TAG}", args: "--device=/dev/kfd --device=/dev/dri --group-add video --cap-add SYS_PTRACE -v=/var/jenkins/:/var/jenkins ${docker_args}") {
                        timeout(time: 2, unit: 'HOURS') {
                            body(cmake_build)
                        }
                    }
                }
            }
        }
    }
}
def rocmtest(m) {
    def builders = [:]
    m.each { e ->
        def label = e.key;
        def action = e.value;
        builders[label] = {
            action(label)
        }
    }
    parallel builders
}

def rocmnodename(name) {
    return "master"
}

def rocmnode(name, body) {
    return { label ->
        rocmtestnode(variant: label, node: rocmnodename(name), body: body)
    }
}

properties([
    parameters([
        booleanParam(name: 'FORCE_DOCKER_IMAGE_BUILD', defaultValue: false)
    ])
])

node() {
    println(params.FORCE_DOCKER_IMAGE_BUILD)
    Boolean imageExists = false
    docker.withRegistry('https://registry.hub.docker.com', 'DOCKER_HUB_CREDS') {
        stage('Check image') {
            env.IMAGE_TAG = sh(script: 'sha256sum **/Dockerfile **/*requirements.txt **/install_prereqs.sh **/rbuild.ini | sha256sum | cut -d " " -f 1', returnStdout: true)
            println(env.IMAGE_TAG)
            imageExists = sh(script: "docker manifest inspect ${DOCKER_IMAGE}:${IMAGE_TAG}", returnStatus: true) == 0
            println(imageExists)
        }
        stage('Build image') {
            if(!imageExists || params.FORCE_DOCKER_IMAGE_BUILD) {
                def builtImage

                try {
                    sh "docker pull ${DOCKER_IMAGE}:latest"
                    builtImage = docker.build("${DOCKER_IMAGE}:${IMAGE_TAG}", "--cache-from ${DOCKER_IMAGE}:latest .")
                } catch(Exception ex) {
                    builtImage = docker.build("${DOCKER_IMAGE}:${IMAGE_TAG}", " --no-cache .")
                }
                println(builtImage)
                builtImage.push("${DOCKER_IMAGE}:${IMAGE_TAG}")
                builtImage.push("${DOCKER_IMAGE}:latest")
            } else {
                // Skip stage so it remains in the visualization
                Utils.markStageSkippedForConditional(STAGE_NAME)
            }
        }
    }
}
rocmtest clang_debug: rocmnode('cdna') { cmake_build ->
    stage('hipRTC Debug') {
        println("hipRTC Debug stage body")
    }
}, clang_release: rocmnode('mi100+') { cmake_build ->
    stage('Hip Clang Release') {
        println("Hip Clang Release body")
    }
}
