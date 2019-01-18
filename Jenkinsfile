import java.text.SimpleDateFormat

pipeline {
  options {
    buildDiscarder logRotator(numToKeepStr: '5')
    disableConcurrentBuilds()
  }
  agent {
    kubernetes {
      cloud "aegjxnode-build"
      label "aegjxnode-build"
      serviceAccount "build"
      yamlFile "KubernetesPod.yaml"
    }
  }
  environment {
    image = "tonygilkerson/aegjxnode"
    project = "aegjxnode"
    namespace = "aegjxnode-build"
    domain = "127.0.0.1.nip.io"
    cmAddr = "cm-chartmuseum.charts:8080"
    tagBeta = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    tagRelease = "${env.BUILD_NUMBER}"
    chartName = "${project}-${env.BUILD_NUMBER}-${env.BRANCH_NAME}"
    deploymentName = "${project}-${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${project}"
    addr = "${project}-${env.BUILD_NUMBER}-${env.BRANCH_NAME}.${domain}"
  }
  stages {
    //
    //  Run static unit tests
    //  dockr push beta image
    //
    stage("static-tests") {
      steps {
        container("nodejs") {
          sh "npm install"
          sh "CI=true DISPLAY=:99 npm test"
        }
        container("docker") {
          sh """docker image build -t ${image}:${tagBeta} ."""
          withCredentials([usernamePassword(
              credentialsId: "docker",
              usernameVariable: "USER",
              passwordVariable: "PASS"
          )]) {
              sh """docker login -u $USER -p $PASS"""
          }
          sh """docker image push ${image}:${tagBeta}"""
        }
      }
    }
    //
    //  Install helm chart
    //  Run functional test
    //  Always cleanup with helm delete
    //
    stage("func-test") {
      steps {
        container("helm") {
          sh """helm upgrade \
              ${chartName.toLowerCase()} \
              charts/${project} -i \
              --namespace ${namespace} \
              --tiller-namespace ${namespace} \
              --set image.tag=${tagBeta} \
              --set image.repository=${image} \
              --set ingress.host=${addr.toLowerCase()}"""
        }
        container("kubectl") {
          sh """kubectl -n ${namespace} rollout status deployment \
              ${deploymentName.toLowerCase()}"""
        }
        // todo run functional tests here
      }
      post {
        always {
          container("helm") {
            sh """helm delete ${chartName.toLowerCase()} \
                --tiller-namespace ${namespace} \
                --purge"""
          }
        }
      }
    }
    //
    //  Docker push prod tag
    //  Publish new version of chart
    //
    stage("release") {
      when {
          branch "master"
      }
      steps {
        container("docker") {
          sh """docker pull ${image}:${tagBeta}"""
          sh """docker image tag ${image}:${tagBeta} ${image}:${tagRelease}"""
          sh """docker image tag ${image}:${tagBeta} ${image}:latest"""
          withCredentials([usernamePassword(
              credentialsId: "docker",
              usernameVariable: "USER",
              passwordVariable: "PASS")]) {

            sh """docker login -u $USER -p $PASS"""
          }
          sh """docker image push ${image}:${tagRelease}"""
          sh """docker image push ${image}:latest"""

        }
        container("helm") {
          script {
            withCredentials([usernamePassword(credentialsId: "chartmuseum", usernameVariable: "USER", passwordVariable: "PASS")]) {

              // Fail if chart version already exists
              chartYaml = readYaml file: "charts/${project}/Chart.yaml"
              out = sh returnStdout: true, script: "curl -u $USER:$PASS http://${cmAddr}/api/charts/${project}/${chartYaml.version}"
              if (!out.contains("error")) {
                  error "Did you forget to increment the Chart version?"
              }

              // Replace tag
              valuesYaml = readYaml file: "charts/${project}/values.yaml"
              valuesYaml.image.tag = "${tagRelease}"
              sh "rm -f charts/${project}/values.yaml"
              writeYaml file: "charts/${project}/values.yaml", data: valuesYaml

              // Package chart and publish to chart repo
              sh "helm package charts/${project}"
              packageName = sh(returnStdout: true, script: "ls ${project}*").trim()
              sh """curl -u $USER:$PASS --data-binary "@${packageName}" http://${cmAddr}/api/charts"""

            }
          }

        }
      }
    }
  }
}
