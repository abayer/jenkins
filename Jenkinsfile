// Generic is the label I'm using on my test setup
node('generic') {
  stage "Checkout source"

  checkout scm

  stage "Build and test"
  timeout(time: 180, unit: 'MINUTES') {
    withMavenEnv(["-Xmx1536m", "-Xms512m", "-XX:MaxPermSize=1024m"]) {
      sh "mvn -Pdebug -U clean install -Dmaven.test.failure.ignore=true -Dconcurrency=1 -V -B -Dmaven.repo.local=${pwd()}/.repository"
    }
  }

  stage "Archive artifacts and test results"
  archive includes: "**/target/*.jar, **/target/*.war, **/target/*.hpi"
  step([$class: 'JUnitResultArchiver', healthScaleFactor: 20.0, testResults: '**/target/surefire-reports/*.xml'])

}

void withMavenEnv(List envVars = [], def body) {
  String mvntool = tool name: "mvn3.3.3", type: 'hudson.tasks.Maven$MavenInstallation'
  String jdktool = tool name: "jdk7u80", type: 'hudson.model.JDK'

  List mvnEnv = ["PATH+MVN=${mvntool}/bin", "PATH+JDK=${jdktool}/bin", "JAVA_HOME=${jdktool}", "MAVEN_HOME=${mvntool}"]
  mvnEnv.addAll(envVars)
  withEnv(mvnEnv) {
    body.call()
  }
}