#!groovy

def tryStep(String message, Closure block, Closure tearDown = null) {
    try {
        block();
    }
    catch (Throwable t) {
        slackSend message: "${env.JOB_NAME}: ${message} failure ${env.BUILD_URL}", channel: '#ci-channel', color: 'danger'

        throw t;
    }
    finally {
        if (tearDown) {
            tearDown();
        }
    }
}


node {
    stage("Checkout") {
        checkout scm
    }


    stage("Build image") {
        tryStep "build", {
            def image = docker.build("build.datapunt.amsterdam.nl:5000/datapunt/mapserver-public:${env.BUILD_NUMBER}")
            image.push()
        }
        tryStep "build private", {
            sh "docker build -f Dockerfile_private build.datapunt.amsterdam.nl:5000/datapunt/mapserver-private:${env.BUILD_NUMBER}" &&
                    "docker tag build.datapunt.amsterdam.nl:5000/datapunt/mapserver-private:${env.BUILD_NUMBER}" &&
                    "docker push build.datapunt.amsterdam.nl:5000/datapunt/mapserver-private:${env.BUILD_NUMBER}"
        }
    }
}

String BRANCH = "${env.BRANCH_NAME}"

if (BRANCH == "master") {

    node {
        stage('Push acceptance image') {
            tryStep "image tagging", {
                def image = docker.image("build.datapunt.amsterdam.nl:5000/datapunt/mapserver-public:${env.BUILD_NUMBER}")
                image.pull()
                image.push("acceptance")
            }
        }

        stage('Push acceptance image') {
            tryStep "image tagging", {
                def image = docker.image("build.datapunt.amsterdam.nl:5000/datapunt/mapserver-private:${env.BUILD_NUMBER}")
                image.pull()
                image.push("acceptance")
            }
        }
    }

    node {
        stage("Deploy to ACC") {
            tryStep "deployment", {
                build job: 'Subtask_Openstack_Playbook',
                parameters: [
                    [$class: 'StringParameterValue', name: 'INVENTORY', value: 'acceptance'],
                    [$class: 'StringParameterValue', name: 'PLAYBOOK', value: 'deploy-mapserver.yml'],
                ]
            }
        }
    }


    stage('Waiting for approval') {
        slackSend channel: '#ci-channel', color: 'warning', message: 'Mapserver is waiting for Production Release - please confirm'
        input "Deploy to Production?"
    }

    node {
        stage('Push production image') {
            tryStep "image tagging", {
                def image = docker.image("build.datapunt.amsterdam.nl:5000/datapunt/mapserver-public:${env.BUILD_NUMBER}")
                image.pull()

                image.push("production")
                image.push("latest")
            }
            tryStep "image tagging private", {
                def image = docker.image("build.datapunt.amsterdam.nl:5000/datapunt/mapserver-private:${env.BUILD_NUMBER}")
                image.pull()

                image.push("production")
                image.push("latest")
            }
        }
    }

    node {
        stage("Deploy") {
            tryStep "deployment", {
                build job: 'Subtask_Openstack_Playbook',
                parameters: [
                    [$class: 'StringParameterValue', name: 'INVENTORY', value: 'production'],
                    [$class: 'StringParameterValue', name: 'PLAYBOOK', value: 'deploy-mapserver.yml'],
                ]
            }
        }
    }
}