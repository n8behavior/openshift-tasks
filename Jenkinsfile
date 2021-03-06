#!groovy

// Run this node on a Maven Slave
// Maven Slaves have JDK and Maven already installed
node('maven') {
  // Define Maven Command. Make sure it points to the correct settings for our
  // Nexus installation. The file nexus_openshift_settings.xml needs to be in the
  // Source Code repository.
  def mvnCmd = "mvn -s ./settings.xml"

  stage('Checkout Source') {
    // Get Source Code from SCM (Git) as configured in the Jenkins Project
    // Next line for inline script, "checkout scm" for Jenkinsfile from Gogs
    git 'https://github.com/n8behavior/openshift-tasks'
    checkout scm
  }

  // The following variables need to be defined at the top level and not inside
  // the scope of a stage - otherwise they would not be accessible from other stages.
  // Extract version and other properties from the pom.xml
  def groupId    = getGroupIdFromPom("pom.xml")
  def artifactId = getArtifactIdFromPom("pom.xml")
  def version    = getVersionFromPom("pom.xml")

  // Using Maven build the war file
  // Do not run tests in this step
  stage('Build war') {
    sh "${mvnCmd} clean package -DskipTests"
  }

  // Using Maven run the unit tests
  stage('Unit Tests') {
    sh "${mvnCmd} test"
  }

  // Using Maven call SonarQube for Code Analysis
  stage('Code Analysis') {
    sh "${mvnCmd} sonar:sonar -DskipTests"
  }

  // Publish the latest war file to Nexus. This needs to go into <nexusurl>/repository/releases.
  // Using the properties from the pom.xml file construct a filename that includes the version number from the pom.xml file
  //
  // You now have two options: Either binary build (streaming the war file into a binary build) or build with external artifacts.
  // If you are unclear which path to follow ask your instructor.
  // 
  // When doing a binary build you are done.
  // 
  // When doing a build with external artifacts (not binary build) you will also need to do the following:
  // - Update the Gogs project openshift-tasks-ocp by editing the .s2i/environment file. This file needs to have a line
  //   WAR_FILE_LOCATION=<actual URL of the war file in nexus>
  // - It is also a good idea to add another line like "BUILD_NUMBER=${BUILD_NUMBER}" to the environment file. Otherwise the
  //   push to Gig/Gogs will fail in case the version number didn't change. ${BUILD_NUMBER} is one of the Jenkins built-in
  //   variables.
  stage('Publish to Nexus') {
    sh "${mvnCmd} deploy -DskipTests"
  }

  // Build the OpenShift Image in OpenShift. 
  // 1. When doing a binary build make sure to rename the file ./target/openshift-tasks.war to ROOT.war before you start the build.
  // 2. When doing a build with external artifacts make sure to use the build configuration pointing to openshift-tasks-ocp
  //    for the .s2i/bin/assemble script to retrieve the war file from the location in the .s2i/environment file.
  // Also tag the image with "TestingCandidate-${version}" - e.g. TestingCandidate-1.5
  stage('Build OpenShift Image') {
    def newTag = "TestingCandidate-${version}"
    echo "New Tag: ${newTag}"

    sh "cp ./target/openshift-tasks.war ./ROOT.war"

    sh "oc project mjs-tasks-dev"
    sh "oc start-build tasks --follow --from-file=./ROOT.war -n mjs-tasks-dev"

    openshiftTag alias: 'false', destStream: 'tasks', destTag: newTag, destinationNamespace: 'mjs-tasks-dev', namespace: 'mjs-tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
  }

  // Deploy the built image to the Development Environment. Pay close attention to WHICH image you are deploying.
  // Make sure it is the one you just tagged in the previous step. You may need to patch the deployment configuration
  // of your application.
  stage('Deploy to Dev') {
    sh "oc project mjs-tasks-dev"
    sh "oc patch dc tasks --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"tasks\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"mjs-tasks-dev\", \"name\": \"tasks:TestingCandidate-$version\"}}}]}}' -n mjs-tasks-dev"

    openshiftDeploy depCfg: 'tasks', namespace: 'mjs-tasks-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyDeployment depCfg: 'tasks', namespace: 'mjs-tasks-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
    openshiftVerifyService namespace: 'mjs-tasks-dev', svcName: 'tasks', verbose: 'false'
  }

  // Run some integration tests (see the openshift-tasks Github Repository README.md for ideas).
  // Once the tests succeed tag the image as ProdReady-${version}
  stage('Integration Test') {
    // TBD
  }

  // Blue/Green Deployment into Production
  // -------------------------------------
  // Next two stages could be one.
  // Make sure to deploy the right version. If green is active then deploy blue, and vice versa.
  // You will need to figure out which application is active and set the target to the other.
  stage('Prep Production Deployment') {
    // TBD
  }
  // Deploy the ProdReady-${version} image. Make sure this is the actual tagged image deployed!
  // Do not activate the new version yet.
  stage('Deploy new Version') {
    // TBD
  }

  // Once approved (input step) switch production over to the new version.
  stage('Switch over to new Version') {
    input "Switch Production?"
    // TBD
  }
}

// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}
def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}
