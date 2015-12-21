#!groovy
// TEST FLAG - to make it easier for me to turn on/off unit tests for speeding up access to later stuff.
def runTests = false

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

        stage "packaging - actually packaging and testing"
        // Working packaging code, separate branch with fixes
        dir('packaging') {
            git branch: 'master', url: 'https://github.com/jenkinsci/packaging.git'
            // Grab the war file from the stash - it goes to war/target/jenkins.war
            unstash "jenkins.war"
            sh "cp war/target/jenkins.war ."

            sh 'docker run --rm -v "`pwd`":/tmp/packaging -w /tmp/packaging jenkins-packaging-builder:0.1 make clean deb rpm suse BRAND=./branding/jenkins.mk BUILDENV=./env/test.mk CREDENTIAL=./credentials/test.mk WAR=jenkins.war'
        }

        withEnv(['HOME='+pwd()]) {
            def img = docker.image('jenkins-packaging-builder:0.1')
            img.inside() {
                sh 'env'
            }
        }

    }
}

stage "Acceptance test harness"

if (runTests) {
// Split the tests up - currently we're splitting into 8 piles to be run concurrently.
    def splits = splitTests([$class: 'CountDrivenParallelism', size: 8])

// Because of limitations in Workflow at this time, we can't just do this via something
// like .collectEntries - we have to jump through some hoops to put together the map of names to
// steps that we pass onward to parallel.
    def branches = [:]
    for (int i = 0; i < splits.size(); i++) {
        def exclusions = splits.get(i);
        branches["split${i}"] = {
            // Run on the generic node for now.
            node('generic') {
                // We need the Maven and Java environments here too.
                withMavenEnv(["JAVA_OPTS=-Xmx1536m -Xms512m -XX:MaxPermSize=1024m",
                              "MAVEN_OPTS=-Xmx1536m -Xms512m -XX:MaxPermSize=1024m"]) {
                    // Get the ATH source.
                    git url: 'git://github.com/jenkinsci/acceptance-test-harness.git', branch: 'master'

                    // Filter out the tests we don't want to run.
                    writeFile file: 'excludes.txt', text: exclusions.join("\n")
                    sh 'cat excludes.txt'

                    // Get the jenkins.war from above that we'll be testing against and
                    // put it in place.
                    unstash "jenkins.war"
                    sh "cp war/target/jenkins.war ."

                    // Run the selected tests within xvnc.
                    wrap([$class: 'Xvnc', takeScreenshot: false, useXauthority: true]) {
                        sh 'mvn clean test -B -Dmaven.test.failure.ignore=true -DforkCount=2'
                    }

                    // And archive the test results once we're done.
                    step([$class: 'JUnitResultArchiver', testResults: 'target/surefire-reports/*.xml'])
                }
            }
        }
    }

// Now, actually launch 'em in parallel!
    parallel branches
} else {
    echo "Skipping ATH..."
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
