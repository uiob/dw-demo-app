node {
  stage 'Checkout'
  checkout scm

  stage 'Configuration'
  sh 'git rev-parse --short HEAD > GIT_COMMIT'
  short_commit = readFile('GIT_COMMIT').trim()
  sh 'echo $PUBLIC_IP > PUBLIC_IP'
  def pom = readMavenPom file:'pom.xml'
  artefactName = "${pom.getArtifactId()}.${pom.getPackaging()}"
  publicIp = readFile('PUBLIC_IP').trim()
  def mvnHome = tool name: 'maven3', type: 'hudson.tasks.Maven$MavenInstallation'
  sh 'rm GIT_COMMIT PUBLIC_IP'

  stage 'Build'
  sh "${mvnHome}/bin/mvn clean compile"

  stage 'Unit Tests'
  sh "${mvnHome}/bin/mvn test"
  step([$class: 'JUnitResultArchiver', testResults: 'target/surefire-reports/*.xml'])

  stage 'Package'
  sh "${mvnHome}/bin/mvn package"
  stash name: 'app', includes: "target/${artefactName}"
  stash name: 'dockerfile', includes: "Dockerfile"
  stash name: 'config', includes: "hello-world.yml"

  stage 'Integration Tests'
  sh "${mvnHome}/bin/mvn verify"
}

node('docker'){
  stage 'Docker build'
  unstash 'config'
  unstash 'dockerfile'
  unstash 'app'
  dockerImage = "demo-app:${short_commit}"
  image = docker.build("${dockerImage}")
  docker.withRegistry('http://localhost:5000', '') {
    image.push("${short_commit}")
  }

  stage 'Launching Docker image for manual validation'
  container = image.run('-P')
  sh "docker port ${container.id} 8080 | cut -d':' -f2 > EXPOSED_PORT"
  exposed_port = readFile('EXPOSED_PORT').trim()
  sh 'rm EXPOSED_PORT'

}


try {
  timeout(time:5, unit:'DAYS') {
    input message: "http://${publicIp}:${exposed_port}. Is it ok?", ok: 'Publish it'
  }
} finally {
  node('docker') { container.stop() }
}

node('docker') {
  stage 'Deploy'
  docker.withRegistry('http://localhost:5000', '') {
    image.push('latest')
  }
}
