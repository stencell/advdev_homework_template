// Set your project Prefix using your GUID
def prefix      = "8370"

// Set variable globally to be available in all stages
// Set Maven command to always include Nexus Settings
def mvnCmd      = "mvn -s ./nexus_openshift_settings.xml"
// Set Development and Production Project Names
def devProject  = "${prefix}-tasks-dev"
def prodProject = "${prefix}-tasks-prod"
// Set the tag for the development image: version + build number
def devTag      = "0.0-0"
// Set the tag for the production image: version
def prodTag     = "0.0"
def destApp     = "tasks-green"
def activeApp   = ""

pipeline {
  agent {
    // Using the Jenkins Agent Pod that we defined earlier
    label "maven-appdev"
  }
  stages {
    // Checkout Source Code and calculate Version Numbers and Tags
    stage('Checkout Source') {
      steps {
        // TBD: Get code from protected Git repository
        git credentialsId: '2da8352b-3e67-4dc3-95d9-dc790b602a1f', url: 'http://gogs-8370-gogs.apps.cluster-scl-ilt.scl-ilt.openshift.opentlc.com/CICDLabs/openshift-tasks-private.git'
        // git 'https://github.com/stencell/advdev_homework_template.git'

       script {
          def pom = readMavenPom file: 'pom.xml'
          def version = pom.version

          // TBD: Set the tag for the development image: version + build number.
          // Example: def devTag  = "0.0-0"
          devTag = "${version}-" + currentBuild.number

          // TBD: Set the tag for the production image: version
          // Example: def prodTag = "0.0"
          prodTag = "${version}"

        }
      }
    }

    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build War File') {
      steps {
        echo "Building version ${devTag}"
            sh "${mvnCmd} clean package -DskipTests=true"
      }
    }

    // Using Maven run the unit tests
    stage('Unit Tests') {
      steps {
        echo "Running Unit Tests"
        // sh "${mvnCmd} test"

        // step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
      }
    }

    //Using Maven call SonarQube for Code Analysis
    stage('Code Analysis') {
      steps {
        echo "Running Code Analysis"
            sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube.${prefix}-sonarqube.svc.cluster.local:9000/ -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"

        // TBD

      }
    }

    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      steps {
        echo "Publish to Nexus"
            sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.${prefix}-nexus.svc.cluster.local:8081/repository/releases"

        // TBD

      }
    }

    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      steps {
        echo "Building OpenShift container image tasks:${devTag}"

        // TBD: Start binary build in OpenShift using the file we just published.
        // Either use the file from your
        // workspace (filename to pass into the binary build
        // is openshift-tasks.war in the 'target' directory of
        // your current Jenkins workspace).
        // OR use the file you just published into Nexus:
        // "--from-file=http://nexus3.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war"


        // TBD: Tag the image using the devTag.
        script {
            openshift.withCluster() {
                openshift.withProject("${devProject}") {
                openshift.selector("bc", "tasks").startBuild("--from-file=./target/openshift-tasks.war", "--wait=true")

            openshift.tag("tasks:latest", "tasks:${devTag}")
            }
          }
        }
      }
    }

    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      steps {
        echo "Deploy container image to Development Project"

        // TBD: Deploy the image
        // 1. Update the image on the dev deployment config
        // 2. Update the config maps with the potentially changed properties files
       // 3. Reeploy the dev deployment
       // 4. Wait until the deployment is running
       //    The following code will accomplish that by
       //    comparing the requested replicas
       //    (rc.spec.replicas) with the running replicas
       //    (rc.status.readyReplicas)
       //

      // def dc = openshift.selector("dc", "tasks").object()
      // def dc_version = dc.status.latestVersion
      // def rc = openshift.selector("rc", "tasks-${dc_version}").object()

      // echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
      // while (rc.spec.replicas != rc.status.readyReplicas) {
      //   sleep 5
      //   rc = openshift.selector("rc", "tasks-${dc_version}").object()
      // }
        script {
            openshift.withCluster() {
                openshift.withProject("${devProject}") {
                openshift.set("image", "dc/tasks", "tasks=image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${devTag}")

                openshift.selector('configmap', 'tasks-config').delete()
                def configmap = openshift.create('configmap', 'tasks-config', '--from-file=./configuration/application-users.properties', '--from-file=./configuration/application-roles.properties' )

                openshift.selector("dc", "tasks").rollout().latest();

                def dc = openshift.selector("dc", "tasks").object()
                def dc_version = dc.status.latestVersion
                def rc = openshift.selector("rc", "tasks-${dc_version}").object()

                echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
                while (rc.spec.replicas != rc.status.readyReplicas) {
                    sleep 5
                    rc = openshift.selector("rc", "tasks-${dc_version}").object()
          }
        }
      }
    }
  }
}

    // Run Integration Tests in the Development Environment.
    stage('Integration Tests') {
      steps {
        echo "Running Integration Tests"
        script {
          def status = "000"

          // Create a new task called "integration_test_1"
          echo "Creating task"
          status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1").trim()
          echo "Status: " + status
          if (status != "201") {
              error 'Integration Create Test Failed!'
          }
          // The next bit works - but only after the application
          // has been deployed successfully
          // status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1").trim()
          // echo "Status: " + status
          // if (status != "201") {
          //     error 'Integration Create Test Failed!'
          // }

          echo "Retrieving tasks"
          // TBD: Implement check to retrieve the task
          status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Accept: application/json' -X GET http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
          if (status != "200") {
              error 'Integration Get Test Failed!'
          }

          echo "Deleting tasks"
          // TBD: Implement check to delete the task
          status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -X DELETE http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
          if (status != "204") {
              error 'Integration Create Test Failed!'
          }

        }
      }
    }

    // Copy Image to Nexus Docker Registry
    stage('Copy Image to Nexus Docker Registry') {
      steps {
        echo "Copy image to Nexus Docker Registry"
        // TBD. Use skopeo to copy
        script {
        // OpenShift 4
        sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:950940ec-c2b4-446d-aeda-e345d0ef0e4c docker://image-registry.openshift-image-registry.svc.cluster.local:5000/${devProject}/tasks:${devTag} docker://nexus3-registry.${prefix}-nexus.svc.cluster.local:5000/tasks:${devTag}"

        // OpenShift 3
        // sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin123 docker://docker-registry.default.svc.cluster.local:5000/${devProject}/tasks:${devTag} docker://nexus3-registry.${prefix}-nexus.svc.cluster.local:5000/tasks:${devTag}"

        // Tag the built image with the production tag.
        openshift.withCluster() {
          openshift.withProject("${prodProject}") {
            openshift.tag("${devProject}/tasks:${devTag}", "${devProject}/tasks:${prodTag}")
          }
        }
      }
      }
    }

    // Blue/Green Deployment into Production
    // -------------------------------------
    // Do not activate the new version yet.
    stage('Blue/Green Production Deployment') {
      steps {
        echo "Blue/Green Deployment"

        // TBD: 1. Determine which application is active
        //      2. Update the image for the other application
        //      3. Deploy into the other application
        //      4. Update Config maps for other application
        //      5. Wait until application is running
        //         See above for example code
        script {
        openshift.withCluster() {
          openshift.withProject("${prodProject}") {
            activeApp = openshift.selector("route", "tasks").object().spec.to.name
            if (activeApp == "tasks-green") {
              destApp = "tasks-blue"
            }
            echo "Active Application:      " + activeApp
            echo "Destination Application: " + destApp

            // Update the Image on the Production Deployment Config
            def dc = openshift.selector("dc/${destApp}").object()

            // OpenShift 4
            dc.spec.template.spec.containers[0].image="image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${prodTag}"
            // OpenShift 3
            // dc.spec.template.spec.containers[0].image="docker-registry.default.svc:5000/${devProject}/tasks:${prodTag}"

            openshift.apply(dc)

            // Update Config Map in change config files changed in the source
            openshift.selector("configmap", "${destApp}-config").delete()
            def configmap = openshift.create("configmap", "${destApp}-config", "--from-file=./configuration/application-users.properties", "--from-file=./configuration/application-roles.properties" )

            // Deploy the inactive application.
            openshift.selector("dc", "${destApp}").rollout().latest();

            // Wait for application to be deployed
            def dc_prod = openshift.selector("dc", "${destApp}").object()
            def dc_version = dc_prod.status.latestVersion
            def rc_prod = openshift.selector("rc", "${destApp}-${dc_version}").object()
            echo "Waiting for ${destApp} to be ready"
            while (rc_prod.spec.replicas != rc_prod.status.readyReplicas) {
              sleep 5
              rc_prod = openshift.selector("rc", "${destApp}-${dc_version}").object()
            }
          }
        }
      }
      }
    }

    stage('Switch over to new Version') {
      steps {
        // TBD: Stop for approval
        input "Switch Production?"


        echo "Executing production switch"
        // TBD: After approval execute the switch
        script {
        openshift.withCluster() {
          openshift.withProject("${prodProject}") {
            def route = openshift.selector("route/tasks").object()
            route.spec.to.name="${destApp}"
            openshift.apply(route)
      }
    }
  }
}
    }
  }
}