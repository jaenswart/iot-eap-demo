#!/usr/bin/env groovy

node {
    stage 'Git checkout'
    echo 'Checking out git repository'
    git url: 'https://github.com/jamesfalkner/iot-eap-demo'

    stage 'Build microservices with maven'
    echo 'Building project'
    def mvnHome = tool 'M3'
    def javaHome = tool 'jdk8'

    stage 'Build image and deploy in Dev'
    echo 'Building docker image and deploying to Dev'
    buildProject('iot-eap-demo-dev')

    stage 'Automated tests'
    echo 'This stage simulates automated tests'
    sh "${mvnHome}/bin/mvn -B -Dmaven.test.failure.ignore clean package verify"

    stage 'Deploy to QA'
    echo 'Deploying to QA'
    deployProject('iot-eap-demo-dev', 'iot-eap-demo-qa')

    stage 'Wait for approval'
    input 'Aprove to production?'

    stage 'Deploy to production'
    echo 'Deploying to production'
    deployProject('iot-eap-demo-dev', 'iot-eap-demo')
}

// Creates a Build and triggers it
def buildProject(String project){
    projectSet(project)
    sh "oc delete bc --all"
    sh "oc new-app --name=iot-device-service https://github.com/openshift/nodejs-ex"
    sh "oc new-build . --name=iot-frontend -l app=iot-frontend --image=jboss-eap70-openshift:1.4 --strategy=source || echo 'Build exists'"
    sh "oc logs -f bc/iot-frontend"
    appDeploy()
}

// Tag the ImageStream from an original project to force a deployment
def deployProject(String origProject, String project){
    projectSet(project)
    sh "oc policy add-role-to-user system:image-puller system:serviceaccount:${project}:default -n ${origProject}"
    sh "oc tag ${origProject}/iot-frontend:latest ${project}/iot-frontend:latest"
    sh "oc tag ${origProject}/iot-device-service:latest ${project}/iot-device-service:latest"
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
    sh "oc delete dc/iot-frontend || echo 'Application deployment exists'"
    sh "oc new-app iot-frontend -l app=iot-frontend,hystrix.enabled=true || echo 'Application already Exists'"
    sh "oc new-app iot-device-service -l app=iot-device-service,hystrix.enabled=true || echo 'Application service already Exists'"
    sh "oc expose service iot-frontend || echo 'Service already exposed'"
    sh 'oc patch dc/iot-frontend -p \'{"spec":{"triggers":[]}}\''
    sh 'oc patch dc/iot-frontend -p \'{"spec":{"template":{"spec":{"containers":[{"name":"iot-frontend","ports":[{"containerPort": 8778,"name":"jolokia"}]}]}}}}\''
    sh 'oc patch dc/iot-frontend -p \'{"spec":{"template":{"spec":{"containers":[{"name":"iot-frontend","readinessProbe":{"httpGet":{"path":"/","port":8080}}}]}}}}\''
}

