// Generic is the label I'm using on my test setup
node('generic') {

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