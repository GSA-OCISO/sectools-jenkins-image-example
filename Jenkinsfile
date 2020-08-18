//Used as a place holder for the image created
def dockerImage
//The image name you want to use for your build environment/
def buildImageName = "nginx:latest"
//The Docker file we will use to build the image we will Push to our repository
def dockerFile = 'Dockerfile'
//The image name we will Push to our repository
def pushImageName = "627566894399.dkr.ecr.us-east-1.amazonaws.com/sectools-hardened"
//The image tag we will apply to the image we Push to our repository
def pushImageTag = env.TAG_NAME ?: 'latest'
//Setup Docker registry variables
//The ECR registry information 
def ecrRegistryUrl = 'https://627566894399.dkr.ecr.us-east-1.amazonaws.com/sectools-hardened'
//Credential id stored
def ecrRegistryCredID = 'ecr:us-east-1:sectools-aws-jenkins'
//Slack Channel to post to for build status
def slackChannel = "sectools-builds"
 
pipeline {
    //This is where we request a build agent from Jenkins.
    agent {
        //Select an agent node with the labels Centos and small.
        node {
            label 'ubuntu && docker'
        }
    }
 
    stages {
        stage('Build'){
            steps {
                script {
                    //Step 1. Run build commands within container using the image specified in docker.image
                    docker.image("${buildImageName}").inside {
                            //Just an example showing that what you perform in the default directory persists within the workspace.
                            //You can execute any shell scripts or builds tools you see fit.
                            sh "echo hi > src/html/hi.html"
                        }
                    //You can optionally fire up another container if you require "sidecar" containers by adding another docker.image().inside.
                    //Step 2. This is illustrating that changes made Step 2. inside the Container are persistent.
                    //This is due to the workspace being presented as a volume to the Container.
                    sh 'ls -la src/html/hi.html'
                }
            }
        }
        //In this stage we will build the Docker image using the Dockerfile defined in $dockerFile.
        stage('Build Image'){
            //Run a step conditionally. You can restrict by branches and tags.
            //Leaving the tag key empty will build on any tag.  Optionally, you can use BLOB or REGEXP for tag names.
            when {
                anyOf {
                    branch 'master'
                    tag ''
                }
            }
            steps {
                script {
                    /*
                      Build the Docker image using the Docker file name and tag as defined in the associated variables.
                      Record the image id in the dockerImage variable.  It is required during the Docker Push stage.
                      Note: We force remove any intermediate containers and do not use cache.
                        The reason for not using cache is to force Dockerfile steps that install the latest packages from an
                        external repository to always run even if that step has not changes from the perspective of Docker.
                        ie. If you change a packages minor version upstream in the Pipeline but your install steps for this package
                        do not specify minor version the cached image will see this as an already met requirement and not reinstall
                        if caching is enabled.
                    */
                    dockerImage = docker.build("${pushImageName}:${pushImageTag}", "-f ${dockerFile} --no-cache ./")
 
                }
            }
        }
 
        //In this stage we will want to run addition tests.  If your Pipeline makes it to this stage
        //the Build and Docker Build stages can safely be assumed to have completed successfully.
        stage('Test'){
            steps {
                script {
 
                    //Example of starting and entering a container from the image you created.
                    docker.image("${pushImageName}:${pushImageTag}").inside {
                        //Just an example showing that your image does indeed have the file generated in the build step.
                        //This is due to the Docker file copying the src directory to the /usr/share/nginx/html.
                        sh 'ls -la /usr/share/nginx/html/hi.html'
                    }
                }
            }
        }
 
 
        //In this stage we push the Container we built in stage 'Build Container' to the Docker repo.
        stage('Push Image'){
            when {
                anyOf {
                    branch 'master'
                    tag ''
                }
            }
            steps {
                script {
                    docker.withRegistry("${ecrRegistryUrl}", "${ecrRegistryCredID}") {
                        dockerImage.push()
                        //Alternate way to push by image tag
                        //docker.image('demo').push('latest')
                    }
                    
                }
            }
        }
    }
    //After all Pipleline stages are complete
    post {
        //We don't have a slack channel :)
        //Post failure messages to Slack channel defined in $slackChannel
        //failure {
        //    slackSend (channel: "${slackChannel}", color: '#DC143C', message: "Failed: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        // }
        //success {
            //Uncomment to enable sending messages to Slack
            //slackSend (channel: "${slackChannel}", color: '#008000', message: "Success: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        //}
        cleanup {
            //Cleanup the Docker image if it was created during this run.
            sh "docker image ls $pushImageName"
            sh "if [[ `docker images -q $pushImageName:$pushImageTag 2> /dev/null` != '' ]];then docker image rm $pushImageName:$pushImageTag;fi"
            sh "docker image ls $pushImageName"
        }
    }
 
}
