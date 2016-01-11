#!groovy
// TEST FLAG - to make it easier for me to turn on/off unit tests for speeding up access to later stuff.
def runTests = true

// Only keep the 10 most recent builds.
properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator',
                                                          numToKeepStr: '10']]])

// Generic is the label I'm using on my test setup
node('generic') {

    // Add timestamps to logging output.
    wrap([$class: 'TimestamperBuildWrapper']) {

        // First stage is actually checking out the source. Since we're using Multibranch
        // currently, we can use "checkout scm".
        stage "Checkout source"

        // This will not work for standalone Workflow scripts. There, use:
        // git url: 'git://github.com/jenkinsci/jenkins', branch: '2.0'
        checkout scm

        // Now run the actual build.
        stage "Build and test"

        // We're wrapping this in a timeout - if it takes more than 180 minutes, kill it.
        timeout(time: 180, unit: 'MINUTES') {
            // See below for what this method does - we're passing an arbitrary environment
            // variable to it so that JAVA_OPTS and MAVEN_OPTS are set correctly.
            withMavenEnv(["JAVA_OPTS=-Xmx1536m -Xms512m -XX:MaxPermSize=1024m",
                          "MAVEN_OPTS=-Xmx1536m -Xms512m -XX:MaxPermSize=1024m"]) {
                // Actually run Maven!
                // The -Dmaven.repo.local=${pwd()}/.repository means that Maven will create a
                // .repository directory at the root of the build (which it gets from the
                // pwd() Workflow call) and use that for the local Maven repository.
                sh "mvn -Pdebug -U clean install ${runTests ? '-Dmaven.test.failure.ignore=true -Dconcurrency=1' : '-DskipTests'} -V -B -Dmaven.repo.local=${pwd()}/.repository"
            }
        }

        // Once we've built, archive the artifacts and the test results.
        stage "Archive artifacts and test results"

        archive includes: "**/target/*.jar, **/target/*.war, **/target/*.hpi"
        if (runTests) {
            step([$class: 'JUnitResultArchiver', healthScaleFactor: 20.0, testResults: '**/target/surefire-reports/*.xml'])
        }

        // And stash the jenkins.war for the next step
        stash name: "jenkins.war", includes: "war/target/jenkins.war"
    }
}

// Run the packaging build on a node with the "pkg" label.
node('pkg') {
    // Add timestamps to logging output.
    wrap([$class: 'TimestamperBuildWrapper']) {

        // First stage here is getting prepped for packaging.
        stage "packaging - docker prep"

        // Docker environment to build packagings
        dir('packaging-docker') {
            git branch: 'master', url: 'https://github.com/jenkinsci/packaging.git'
            sh 'docker build -t jenkins-packaging-builder:0.1 docker'
        }

        stage "packaging - actually packaging"
        // Working packaging code, separate branch with fixes
        dir('packaging') {
            git branch: 'master', url: 'https://github.com/jenkinsci/packaging.git'
            // Grab the war file from the stash - it goes to war/target/jenkins.war
            unstash "jenkins.war"
            sh "cp war/target/jenkins.war ."

            sh 'docker run --rm -v "`pwd`":/tmp/packaging -w /tmp/packaging jenkins-packaging-builder:0.1 make clean deb rpm suse BRAND=./branding/jenkins.mk BUILDENV=./env/test.mk CREDENTIAL=./credentials/test.mk WAR=jenkins.war'

            archive includes: "target/**/*"
        }

    }
}

stage "Package testing"

if (true) {
// Basic parameters
    String dockerLabel = 'pkg'
// Basic parameters
    String packagingTestBranch = (binding.hasVariable('packagingTestBranch')) ? packagingTestBranch : '2.0-apb'
    String artifactName = (binding.hasVariable('artifactName')) ? artifactName : 'jenkins'
    String jenkinsPort = (binding.hasVariable('jenkinsPort')) ? jenkinsPort : '8080'

// Set up
    String scriptPath = 'packaging-docker/installtests'

    String debfile = "artifact://${env.JOB_NAME}/${env.BUILD_NUMBER}#target/debian/jenkins_2.0_all.deb"
    String rpmfile = "artifact://${env.JOB_NAME}/${env.BUILD_NUMBER}#target/rpm/jenkins-2.0-1.1.noarch.rpm"
    String susefile = "artifact://${env.JOB_NAME}/${env.BUILD_NUMBER}#target/suse/jenkins-2.0-1.2.noarch.rpm"

// Core tests represent the basic supported linuxes, extended tests build out coverage further
    def coreTests = []
    def extendedTests = []
    coreTests[0]=["sudo-ubuntu:14.04",  ["ubuntu-14.04",
            [
                    ["debian.sh", "installers/deb/*.deb"],
                    ["service-check.sh", "${artifactName} ${jenkinsPort}"]
            ]]]
    coreTests[1]=["sudo-centos:6",  ["centos-6",
            [
                    ["centos.sh", "installers/rpm/*.rpm"],
                    ["service-check.sh", "${artifactName} ${jenkinsPort}"]
            ]]]
    coreTests[2]=["sudo-opensuse:13.2",  ["opensuse-13.2",
            [
                    ["suse.sh", "installers/suse/*.rpm"],
                    ["service-check.sh", "${artifactName} ${jenkinsPort}"]
            ]]]
    extendedTests[0]=["sudo-debian:wheezy",  ["debian-wheezy",
            [
                    ["debian.sh", "installers/deb/*.deb"],
                    ["service-check.sh", "${artifactName} ${jenkinsPort}"]
            ]]]
    extendedTests[1]=["sudo-centos:7",  ["centos-7",
            [
                    ["centos.sh", "installers/rpm/*.rpm"],
                    ["service-check.sh", "${artifactName} ${jenkinsPort}"]
            ]]]
    extendedTests[2]=["sudo-ubuntu:15.10",  ["ubuntu-15.10",
            [
                    ["debian.sh", "installers/deb/*.deb"],
                    ["service-check.sh", "${artifactName} ${jenkinsPort}"]
            ]]]

    node(dockerLabel) {
        // Add timestamps to logging output.
        wrap([$class: 'TimestamperBuildWrapper']) {

            sh 'rm -rf workflowlib'
            dir('workflowlib') {
                git branch: packagingTestBranch, url: 'https://github.com/abayer/jenkins-packaging.git'
                flow = load 'workflow/installertest.groovy'
            }


            flow.fetchInstallers(debfile, rpmfile, susefile)

            sh 'rm -rf packaging-docker'
            dir('packaging-docker') {
                git branch: packagingTestBranch, url: 'https://github.com/abayer/jenkins-packaging.git'
            }

            // Build the sudo dockerfiles

            withEnv(['HOME=' + pwd()]) {
                sh 'packaging-docker/docker/build-sudo-images.sh'
            }

            String[] stepNames = ['install', 'servicecheck']

            stage 'Run Core Installation Tests'
            flow.execute_install_testset(scriptPath, coreTests, stepNames)

            stage 'Run Extended Installation Tests'
            flow.execute_install_testset(scriptPath, extendedTests, stepNames)
        }
    }

} else {
    echo "Skipping package tests"
}

// This method sets up the Maven and JDK tools, puts them in the environment along
// with whatever other arbitrary environment variables we passed in, and runs the
// body we passed in within that environment.
void withMavenEnv(List envVars = [], def body) {
    // The names here are currently hardcoded for my test environment. This needs
    // to be made more flexible.
    // Using the "tool" Workflow call automatically installs those tools on the
    // node.
    String mvntool = tool name: "mvn3.3.3", type: 'hudson.tasks.Maven$MavenInstallation'
    String jdktool = tool name: "jdk7u80", type: 'hudson.model.JDK'

    // Set JAVA_HOME, MAVEN_HOME and special PATH variables for the tools we're
    // using.
    List mvnEnv = ["PATH+MVN=${mvntool}/bin", "PATH+JDK=${jdktool}/bin", "JAVA_HOME=${jdktool}", "MAVEN_HOME=${mvntool}"]

    // Add any additional environment variables.
    mvnEnv.addAll(envVars)

    // Invoke the body closure we're passed within the environment we've created.
    withEnv(mvnEnv) {
        body.call()
    }
}

