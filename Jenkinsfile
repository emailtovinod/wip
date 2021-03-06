def devProject = "test1-005"
def uatProject = "test2-005"
def prodProject = "test3-005"
def appName = "spring-boot-hello-world"

node('maven') {   

    def WORKSPACE = pwd()
    env.KUBECONFIG = pwd() + "/.kubeconfig"

   stage 'Checkout'

       checkout scm

   stage 'Maven Build'

        try {
            sh """
            set +x
            echo "workspace:" ${WORKSPACE}
            mvn build-helper:parse-version versions:set -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.incrementalVersion}-\${BUILD_NUMBER}
            mvn clean install
            """

            step([$class: 'ArtifactArchiver', artifacts: '**/target/*.war, **/target/*.jar', fingerprint: true])
        }
        catch(e) {
            currentBuild.result = 'FAILURE'
            throw e
        }
        finally {
            processStageResult()
        }

      stage "OpenShift Dev Build"

        version = parseVersion("${WORKSPACE}/pom.xml")

        login()

        sh """
         set +x
         currentOutputName=\$(oc get bc ${appName} -n ${devProject} --template=${appName})
         newImageName=\${currentOutputName%:*}:${version}
         oc patch bc ${appName} -n ${devProject} -p "{ \\"spec\\": { \\"output\\": { \\"to\\": { \\"name\\": \\"\${newImageName}\\" } } } }"
         mkdir -p ${WORKSPACE}//target/s2i-build/deployments
         cp ${WORKSPACE}//target/*.war ${WORKSPACE}//target/s2i-build/deployments/
         echo "workspace:" ${WORKSPACE}
         echo "buildnumber:"${BUILD_NUMBER}
         oc start-build ${appName} -n ${devProject} --follow=true --wait=true --from-dir="${WORKSPACE}//target/s2i-build"
         echo "buildnumber2:"${BUILD_NUMBER}
       """

      stage "Dev Deployment"

        login()

        deployApp(appName, devProject, version)

        validateDeployment(appName,devProject)

      stage "Promote to UAT"

        login()

        sh """
          set +x
          echo "Promoting application to UAT Environment"
          oc tag ${devProject}/${appName}:${version} ${uatProject}/${appName}:${version}
          # Sleep for a few moments
          sleep 5
        """

        deployApp(appName, uatProject, version)

        validateDeployment(appName,uatProject)

    stage "Acceptance Checking"

        acceptanceCheck(appName, uatProject)
}

    stage "Promote to Production"

      input "Do you want to promote the ${appName} to Production?"

node('maven') {

    def WORKSPACE = pwd()
    
    env.KUBECONFIG = pwd() + "/.kubeconfig"

      login()

      sh """
        set +x
        echo "Promoting application to Prod Environment"
        oc tag ${uatProject}/${appName}:${version} ${prodProject}/${appName}:${version}
        # Sleep for a few moments
        sleep 5
      """

      deployApp(appName, prodProject, version)

      validateDeployment(appName,prodProject)


}

def processStageResult() {

    if (currentBuild.result != null) {
        sh "exit 1"
    }
}

def login() {
    sh """
       set +x
       oc login --certificate-authority=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt --token=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) https://kubernetes.default.svc.cluster.local >/dev/null 2>&1 || echo 'OpenShift login failed'
       """
}

def parseVersion(String filename) {
  def matcher = readFile(filename) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}

def deployApp(appName, namespace, version) {
            sh """
          set +x
          newDeploymentImageName=${appName}:${version}
          imageReference=\$(oc get is ${appName} -n ${namespace} -o jsonpath="{.status.tags[?(@.tag==\\"${version}\\")].items[*].dockerImageReference}")
          #oc patch dc/${appName} -n ${namespace} -p "{\\"spec\\":{\\"template\\":{\\"spec\\":{\\"containers\\":[{\\"name\\":\\"${appName}\\",\\"image\\": \\"\${imageReference}\\" } ]}}, \\"triggers\\": [ { \\"type\\": \\"ImageChange\\", \\"imageChangeParams\\": { \\"containerNames\\": [ \\"${appName}\\" ], \\"from\\": { \\"kind\\": \\"ImageStreamTag\\", \\"name\\": \\"\${newDeploymentImageName}\\" } } } ] }}"
          oc patch dc/${appName} -n ${namespace} -p "{\\"spec\\":{\\"template\\":{\\"spec\\":{\\"containers\\":[{\\"name\\":\\"${appName}\\",\\"image\\": \\"\${imageReference}\\" } ]}}, \\"triggers\\":  [ ] }}"
          #oc deploy ${appName} -n ${namespace} --latest
          oc rollout latest dc/${appName} -n ${namespace}
          # Sleep for a few moments
          sleep 5
        """


}

def acceptanceCheck(String appName, String namespace) {

    sh """
      set +x
      COUNTER=0
      DELAY=5
      MAX_COUNTER=30
      echo "Running Acceptance Check of ${appName} in project ${namespace}"
     set +e
      while [ \$COUNTER -lt \$MAX_COUNTER ]
      do
        RESPONSE=\$(curl -s -o /dev/null -w '%{http_code}\\n' http://${appName}-${namespace}.cloudapps.example.com/hello)
        if [ \$RESPONSE -eq 200 ]; then
            echo
            echo "Application Verified"
            break
        fi
        if [ \$COUNTER -eq \$MAX_COUNTER ]; then
          echo "Max Validation Attempts Exceeded. Failed Verifying Application Deployment..."
          exit 1
        fi
        sleep \$DELAY
      done
      set -e
      """

}

def validateDeployment(String dcName, String namespace) {

    sh """
      set +x
      COUNTER=0
      DELAY=10
      MAX_COUNTER=30
      echo "Validating deployment of ${dcName} in project ${namespace}"
      LATEST_DC_VERSION=\$(oc get dc ${dcName} -n ${namespace} --template='{{ .status.latestVersion }}')
      RC_NAME=${dcName}-\${LATEST_DC_VERSION}
      set +e
      while [ \$COUNTER -lt \$MAX_COUNTER ]
      do
        RC_ANNOTATION_RESPONSE=\$(oc get rc -n ${namespace} \$RC_NAME --template="{{.metadata.annotations}}")
        echo "\$RC_ANNOTATION_RESPONSE" | grep openshift.io/deployment.phase:Complete >/dev/null 2>&1
        if [ \$? -eq 0 ]; then
          echo "Deployment Succeeded!"
          break
        fi
        echo "\$RC_ANNOTATION_RESPONSE" | grep -E 'openshift.io/deployment.phase:Failed|openshift.io/deployment.phase:Cancelled' >/dev/null 2>&1
        if [ \$? -eq 0 ]; then
          echo "Deployment Failed"
          exit 1
        fi
        if [ \$COUNTER -lt \$MAX_COUNTER ]; then
          echo -n "."
          COUNTER=\$(( \$COUNTER + 1 ))
        fi
        if [ \$COUNTER -eq \$MAX_COUNTER ]; then
          echo "Max Validation Attempts Exceeded. Failed Verifying Application Deployment..."
          exit 1
        fi
        sleep \$DELAY
      done
      set -e
    """
}

