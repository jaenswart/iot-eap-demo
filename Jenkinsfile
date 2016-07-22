#!/usr/bin/env groovy

node {
    stage 'Git checkout'
    echo 'Checking out git repository'
    git url: 'https://github.com/jamesfalkner/iot-eap-demo'

    stage 'Build project with Maven'
    echo 'Building project'
    def mvnHome = tool 'M3'
    def javaHome = tool 'jdk8'
    sh "${mvnHome}/bin/mvn package"

    stage 'Build image and deploy in Dev'
    echo 'Building docker image and deploying to Dev'
    buildProject('iot-eap-demo-dev')

    stage 'Automated tests'
    echo 'This stage simulates automated tests'
    sh "${mvnHome}/bin/mvn -B -Dmaven.test.failure.ignore verify"

    stage 'Deploy to QA'
    echo 'Deploying to QA'
    deployProject('iot-eap-demo-dev', 'iot-eap-demo-qa')

    stage 'Wait for approval'
    input 'Aprove to production?'

    stage 'Deploy to production'
    echo 'Deploying to production'
    deployProject('iot-eap-demo-dev', 'iot-eap-demo-prod')
}

// Creates a Build and triggers it
def buildProject(String project){
    projectSet(project)
    sh "oc new-build --binary --name=iot-eap-demo -l app=iot-eap-demo || echo 'Build exists'"
    sh "oc start-build iot-eap-demo --from-dir=. --follow"
    appDeploy()
}

// Tag the ImageStream from an original project to force a deployment
def deployProject(String origProject, String project){
    projectSet(project)
    sh "oc policy add-role-to-user system:image-puller system:serviceaccount:${project}:default -n ${origProject}"
    sh "oc tag ${origProject}/iot-eap-demo:latest ${project}/iot-eap-demo:latest"
    appDeploy()
}

// Login and set the project
def projectSet(String project){
    //Use a credential called openshift-dev
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'openshift-dev', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
        sh "oc login --insecure-skip-tls-verify=true -u $env.USERNAME -p $env.PASSWORD https://192.168.0.25:8443"
    }
    sh "oc new-project ${project} || echo 'Project exists'"
    sh "oc project ${project}"
}

// Deploy the project based on a existing ImageStream
def appDeploy(){
    sh "oc new-app iot-eap-demo -l app=iot-eap-demo,hystrix.enabled=true || echo 'Aplication already Exists'"
    sh "oc expose service iot-eap-demo || echo 'Service already exposed'"
    sh 'oc patch dc/iot-eap-demo -p \'{"spec":{"template":{"spec":{"containers":[{"name":"iot-eap-demo","ports":[{"containerPort": 8778,"name":"jolokia"}]}]}}}}\''
    sh 'oc patch dc/iot-eap-demo -p \'{"spec":{"template":{"spec":{"containers":[{"name":"iot-eap-demo","readinessProbe":{"httpGet":{"path":"/api/health","port":8080}}}]}}}}\''
}

