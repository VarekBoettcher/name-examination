// FE = front-end code.  JS and VUE.js.

// FE requires a chained build: First buildconfig outputs an intermediate image that's then consumed by a second stage
// (cont..) build that uses it to build the deploy-ready image

// Front-end definitions.  Extra because of the chained build.  "INT" = Intermediate.
def FE_INT_BUILDCFG_NAME ='namex-fe-build'  // TODO: rename build to "namex-fe-int-build"
def FE_INT_IMAGE_NAME = 'namex-front'       // TODO: rename image to "namex-fe-int-image"
def FE_BUILDCFG_NAME ='namex-fe-web'        // TODO: rename build to "namex-fe-build"
def FE_IMAGE_NAME = 'namex-front-caddy'     // TODO: rename image to "namex-fe-image"
def FE_DEPLOYMENT_NAME = 'namex-fe-web-caddy'  // TODO: rename deployment to "namex-fe-deploy"

// **** Note: openshiftVerifyDeploy requires policy to be added:
// oc policy add-role-to-user view system:serviceaccount:<project-prefix>-tools:jenkins -n <project-prefix>-dev
// oc policy add-role-to-user view system:serviceaccount:<project-prefix>-tools:jenkins -n <project-prefix>-test
// oc policy add-role-to-user view system:serviceaccount:<project-prefix>-tools:jenkins -n <project-prefix>-prod

//See https://github.com/jenkinsci/kubernetes-plugin
// The "python3nodejs" has both Python and NodeJS6 so we can use it to build BE and FE
podTemplate(label: 'jenkins-python3nodejs', name: 'jenkins-python3nodejs', serviceAccount: 'jenkins', cloud: 'openshift', containers: [
containerTemplate(
    name: 'jnlp',
    image: '172.50.0.2:5000/openshift/jenkins-slave-python3nodejs',
    resourceRequestCpu: '500m',
    resourceLimitCpu: '1000m',
    resourceRequestMemory: '1Gi',
    resourceLimitMemory: '2Gi',
    workingDir: '/tmp',
    command: '',
    args: '${computer.jnlpmac} ${computer.name}'
    )
])
  // Front-end chain builds
  {
    node('jenkins-python3nodejs') {
        stage('Intermediate build') {
            echo ">>> Building Namex intermediate image..."
                openshiftBuild bldCfg: FE_INT_BUILDCFG_NAME, showBuildLogs: 'true'
            echo ">>> Get Intermediate Image Hash"
            IMAGE_HASH = sh (
                script: 'oc get istag namex-front:latest -o template --template="{{.image.dockerImageReference}}"|awk -F ":" \'{print $3}\'',
                    returnStdout: true).trim()
            echo ">>> IMAGE_HASH: $IMAGE_HASH"
            echo ">>> Intermediate Image Build Complete"

        }
        stage ('Final image build') {
            echo ">>> Building Namex final image..."
            openshiftBuild bldCfg: FE_BUILDCFG_NAME, showBuildLogs: 'true'
            echo ">>> Get Final Image Hash"
            IMAGE_HASH = sh (
                script: 'oc get istag namex-front-caddy:latest -o template --template="{{.image.dockerImageReference}}"|awk -F ":" \'{print $3}\'',
                    returnStdout: true).trim()
            echo ">>> IMAGE_HASH: $IMAGE_HASH"
            openshiftTag destStream: FE_IMAGE_NAME, verbose: 'true', destTag: 'dev', srcStream: FE_IMAGE_NAME, srcTag: "${IMAGE_HASH}"
            sleep 5
            openshiftVerifyDeployment depCfg: FE_DEPLOYMENT_NAME, namespace: 'servicebc-ne-dev', replicaCount: 1, verbose: 'false', verifyReplicaCount: 'false'
            echo ">>> Deployment Complete"
        }
    }
  } // END: Front-end chain builds

// Chained build complete

// *** NOTE: Weird commenting in the bdd section because of a bunch of /* in the commands screwing up the comment nesting

/*
    podTemplate(label: 'jenkins-bdd', name: 'jenkins-bdd', serviceAccount: 'jenkins', cloud: 'openshift', containers: [
    containerTemplate(
        name: 'jnlp',
        image: '172.50.0.2:5000/openshift/jenkins-slave-bddstack',
        resourceRequestCpu: '500m',
        resourceLimitCpu: '1000m',
        resourceRequestMemory: '1Gi',
        resourceLimitMemory: '2Gi',
        workingDir: '/tmp',
        command: '',
        args: '${computer.jnlpmac} ${computer.name}'
        )
    ])  {
        node('jenkins-bdd') {
            stage('Functional Test') {
                //the checkout is mandatory, otherwise functional test would fail
                echo "checking out source"
                heckout scm
                dir('functional-tests') {
                    // retrieving variables from buildConfig
                    TEST_USERNAME = sh (
                    script: 'oc env bc/<name> --list | awk  -F  "=" \'/TEST_USERNAME/{print $2}\'',
                    returnStdout: true).trim()
                    TEST_PASSWORD = sh (
                    script: 'oc env bc/<name> --list | awk  -F  "=" \'/TEST_PASSWORD/{print $2}\'',
                    returnStdout: true).trim()
                    try {
                        sh 'export TEST_USERNAME=${TEST_USERNAME}\nexport TEST_PASSWORD=${TEST_PASSWORD}\n./gradlew --debug --stacktrace chromeHeadlessTest'
                    } finally {
*/
                        //archiveArtifacts allowEmptyArchive: true, artifacts: 'build/reports/**/*'
                        //archiveArtifacts allowEmptyArchive: true, artifacts: 'build/test-results/**/*'
                        //junit 'build/test-results/**/*.xml'
/*
	                }
                }
            }
        }
*/          // *** NOTE: End of wierd commenting


stage('deploy-test') {
  timeout(time: 3, unit: 'DAYS') {
      input message: "Deploy to test?", submitter: 'admin'
  }
  node('master') {
    echo ">>> Send code to test ...."
    openshiftTag destStream: FE_IMAGE_NAME, verbose: 'true', destTag: 'test', srcStream: FE_IMAGE_NAME, srcTag: "${IMAGE_HASH}"
    openshiftVerifyDeployment depCfg: FE_DEPLOYMENT_NAME, namespace: 'servicebc-ne-test', replicaCount: 1, verbose: 'false', verifyReplicaCount: 'false'
    echo ">>> Sending email ...."
    mail (to: 'user@domain', subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) promoted to test", body: "URL: ${env.BUILD_URL}.");
    echo ">>> Stage deploy-test done"
  }

}

/*
stage('deploy-prod') {
  timeout(time: 3, unit: 'DAYS') {
      input message: "Deploy to prod?", submitter: 'admin'
  }
  node('master') {
    echo ">>> Send code to production ...."
    openshiftTag destStream: FE_IMAGE_NAME, verbose: 'true', destTag: 'prodblue', srcStream: FE_IMAGE_NAME, srcTag: 'prod'
    openshiftTag destStream: FE_IMAGE_NAME, verbose: 'true', destTag: 'prod', srcStream: FE_IMAGE_NAME, srcTag: "${IMAGE_HASH}"
    openshiftVerifyDeployment depCfg: FE_DEPLOYMENT_NAME, namespace: '<project-prefix>-prod', replicaCount: 1, verbose: 'false', verifyReplicaCount: 'false'
    echo ">>> Sending email ...."
    mail (to: 'user@domain', subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) promoted to production", body: "URL: ${env.BUILD_URL}.");
    echo ">>> Stage deploy-prod done"
  }
}
*/

