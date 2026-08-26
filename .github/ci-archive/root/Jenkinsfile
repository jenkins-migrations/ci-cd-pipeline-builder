<![CDATA[
// Source: https://github.com/jenkinsci/pipeline-examples (MIT License)
// Test case: Parameters, triggers, and options
pipeline {
  agent any

  parameters {
    booleanParam(defaultValue: true, description: 'Enable feature flag', name: 'flag')
    string(defaultValue: 'hello', description: 'Test string parameter', name: 'SOME_STRING')
    choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target environment')
  }

  triggers {
    cron('@daily')
    pollSCM('H/5 * * * *')
  }

  options {
    buildDiscarder(logRotator(numToKeepStr:'5'))
    disableConcurrentBuilds()
    skipDefaultCheckout(true)
    timeout(time: 10, unit: 'MINUTES')
    timestamps()
  }

  stages {
    stage("Setup") {
      steps {
        checkout scm
        echo "Parameter flag: ${params.flag}"
        echo "Parameter string: ${params.SOME_STRING}"
        echo "Environment: ${params.ENVIRONMENT}"
      }
    }
    stage("Build") {
      steps {
        sh "echo 'Building for ${params.ENVIRONMENT}'"
        sh "test -f README.md || echo 'No README found'"
      }
    }
  }
}
    ]]>