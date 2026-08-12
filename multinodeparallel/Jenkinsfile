<![CDATA[
#!/usr/bin/env groovy
// Source: https://github.com/jenkinsci/pipeline-examples (MIT License)
// Test case: Multi-node parallel builds across different OS platforms
def labels = ['precise', 'trusty', 'windows'] // labels for Jenkins node types we will build on
def builders = [:]
for (x in labels) {
    def label = x // Need to bind the label variable before the closure - can't do 'for (label in labels)'

    // Create a map to pass in to the 'parallel' step so we can fire all the builds at once
    builders[label] = {
      node(label) {
        // build steps that should happen on all nodes go here
        stage('Checkout') {
          checkout scm
        }
        
        stage('Build') {
          if (isUnix()) {
            sh 'make'
          } else {
            bat 'build.bat'
          }
        }
        
        stage('Test') {
          if (isUnix()) {
            sh 'make test'
          } else {
            bat 'test.bat'
          }
        }
        
        stage('Archive') {
          if (isUnix()) {
            archiveArtifacts artifacts: 'build/**/*', fingerprint: true
          } else {
            archiveArtifacts artifacts: 'build\\**\\*', fingerprint: true
          }
        }
      }
    }
}

parallel builders
    ]]>