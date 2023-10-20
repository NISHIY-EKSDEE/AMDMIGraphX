import org.jenkinsci.plugins.pipeline.modeldefinition.Utils

DOCKER_IMAGE = 'rocm/migraphx-ci-ubuntu'

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
            gitStatusWrapper(credentialsId: "${env.status_wrapper_creds}", gitHubContext: "Jenkins - ${variant}", account: 'ROCmSoftwarePlatform', repo: 'AMDMIGraphX') {
                withCredentials([usernamePassword(credentialsId: 'docker_test_cred', passwordVariable: 'DOCKERHUB_PASS', usernameVariable: 'DOCKERHUB_USER')]) {
                    sh "echo $DOCKERHUB_PASS | docker login --username $DOCKERHUB_USER --password-stdin"
                    pre()
                    sh "docker pull ${DOCKER_IMAGE}:${env.IMAGE_TAG}"
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
    def rocmtest_name = "(rocmtest || migraphx)"
    def node_name = "${rocmtest_name}"
    if(name == "fiji") {
        node_name = "${rocmtest_name} && fiji";
    } else if(name == "vega") {
        node_name = "${rocmtest_name} && vega";
    } else if(name == "navi21") {
        node_name = "${rocmtest_name} && navi21";
    } else if(name == "mi100+") {
        node_name = "${rocmtest_name} && (gfx908 || gfx90a) && !vm";
    } else if(name == "cdna") {
        node_name = "${rocmtest_name} && (gfx908 || gfx90a || vega20) && !vm";
    } else if(name == "nogpu") {
        node_name = "${rocmtest_name} && nogpu";
    }
    return node_name
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
    Boolean imageExists = false
    withCredentials([usernamePassword(credentialsId: 'docker_test_cred', passwordVariable: 'DOCKERHUB_PASS', usernameVariable: 'DOCKERHUB_USER')]) {
        sh "echo $DOCKERHUB_PASS | docker login --username $DOCKERHUB_USER --password-stdin"
        stage('Check image') {
            checkout scm
            def calculateImageTagScript = """
                shopt -s globstar
                sha256sum **/Dockerfile **/*requirements.txt **/install_prereqs.sh **/rbuild.ini | sha256sum | cut -d " " -f 1
            """
            env.IMAGE_TAG = sh(script: "bash -c '${calculateImageTagScript}'", returnStdout: true).trim()
            imageExists = sh(script: "docker manifest inspect ${DOCKER_IMAGE}:${IMAGE_TAG}", returnStatus: true) == 0
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
                builtImage.push("${IMAGE_TAG}")
                builtImage.push("latest")
            } else {
                echo "Image already exists, skip building available"
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

rocmtest onnx: onnxnode('mi100+') { cmake_build ->
    stage("Onnx runtime") {
        sh '''
            apt install half
            #ls -lR
            md5sum ./build/*.deb
            dpkg -i ./build/*.deb
            env
            cd /onnxruntime && ./build_and_test_onnxrt.sh
        '''
    }
}
