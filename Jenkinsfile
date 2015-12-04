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
        sh "mvn -Pdebug -U clean install -Dmaven.test.failure.ignore=true -Dconcurrency=1 -V -B -Dmaven.repo.local=${pwd()}/.repository"
      }
    }

    // Once we've built, archive the artifacts and the test results.
    stage "Archive artifacts and test results"

    archive includes: "**/target/*.jar, **/target/*.war, **/target/*.hpi"
    step([$class: 'JUnitResultArchiver', healthScaleFactor: 20.0, testResults: '**/target/surefire-reports/*.xml'])

    // And stash the jenkins.war for the next step
    stash name: "jenkins.war", includes: "war/target/jenkins.war"
  }
}

// Run the packaging build on a node with the "pkg" label.
node('pkg') {
  // Add timestamps to logging output.
  wrap([$class: 'TimestamperBuildWrapper']) {

    // First stage here is getting prepped for packaging.
    stage "packaging prep"

    // Right now, we're using my fork of jenkinsci/packaging.git, where I disable MSI/OS X and will be working on
    // getting the Vagrant testing working in something otehr than VirtualBox.
    git url: "git://github.com/abayer/jenkins-packaging.git", branch: "2.0-apb"

    // Grab the war file from the stash - it goes to war/target/jenkins.war
    unstash "jenkins.war"

    // Same sort of environment as for the build above, but add WAR pointing to the war file.
    withMavenEnv(["JAVA_OPTS=-Xmx1536m -Xms512m -XX:MaxPermSize=1024m",
                  "MAVEN_OPTS=-Xmx1536m -Xms512m -XX:MaxPermSize=1024m",
                  "WAR=${pwd()}/war/target/jenkins.war"]) {
      // Now we start building packages.
      stage "build packages"

      // We're wrapping this in a timeout - if it takes more than 30 minutes, kill it.
      timeout(time: 30, unit: 'MINUTES') {
        // Build the packages via make. Builds RHEL/CentOS/Fedora RPM, Debian package, and SUSE RPM.
        sh "make package BRAND=./branding/jenkins.mk BUILDENV=./env/test.mk CREDENTIAL=./credentials/test.mk"

        // The packages get put in the target directory, so grab that.
        archive includes: "target/**/*"
      }

      // Tests won't work on EC2 thanks to VirtualBox not working on EC2, so gotta work on this more later.
      stage "test packages"
      // We're wrapping this in a timeout - if it takes more than 180 minutes, kill it.
      timeout(time: 180, unit: 'MINUTES') {
        sh "make test"
      }

    }
  }
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