// This Jenkinsfile is for User microservice


pipeline {
    agent {
        label 'k8s-slave'
    }
    parameters {
        choice (name: 'buildOnly',
               choices: 'no\nyes',
               description: "Build the applcation only"
        )
        choice (name: 'scanOnly',
               choices: 'no\nyes',
               description: "This will scan the applciation"
        )
        choice (name: 'dockerPush',
               choices: 'no\nyes',
               description: "This will build the app, push to registry"
        )
        choice (name: 'deployToDev',
               choices: 'no\nyes',
               description: "This will deploy the app to Dev Env"
        )
        choice (name: 'deployToTest',
               choices: 'no\nyes',
               description: "This will deploy the app to Test Env"
        )
        choice (name: 'deployToStage',
               choices: 'no\nyes',
               description: "This will deploy the app to Stage Env"
        )
        choice (name: 'deployToProd',
               choices: 'no\nyes',
               description: "This will deploy the app to Prod Env"
        )
    }
    tools {
        maven 'Maven-3.8.8'
        jdk 'JDK-17'
    }
    environment {
        APPLICATION_NAME = "user"
        // https://www.jenkins.io/doc/pipeline/steps/pipeline-utility-steps/#readmavenpom-read-a-maven-project-file
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        SONAR_URL = "http://35.196.148.247:9000"
        SONAR_TOKEN = credentials('sonar_creds')
        DOCKER_HUB = "docker.io/i27k8s10"
        DOCKER_CREDS = credentials('docker_creds')
        // DOCKER_APPLICATION_NAME = "i27k8s10"
        // DOCKER_HOST_IP = "1.2.3.4"
    }
    stages {
        stage ('Build') {
            when {
                anyOf {
                    expression {
                        params.buildOnly == 'yes'
                    }
                }
            }
            // This step will take care of building the application
            steps {
                script {
                    buildApp().call()
                }
            }
        }
        // stage ('Unit Tests') {
        //     steps {
        //         echo "Executing Unit tests for ${env.APPLICATION_NAME} Application"
        //         sh 'mvn test'
        //     }
        //     post {
        //         always { //success
        //             junit 'target/surefire-reports/*.xml'
        //             jacoco execPattern: 'target/jacoco.exec'
        //         }
        //     }
        // }
        stage ('Sonar') {
            when {
                anyOf {
                    expression {
                        params.scanOnly == 'yes'
                    }
                }
            }
            steps {
                // Code Quality needs to be implemented 
                echo "Starting Sonar Scans with Quality Gates"
                // before we go to next step, install sonarqube plugin 
                // next goto manage jenkins > configure > sonarqube > give url and token for sonarqube
                withSonarQubeEnv('SonarQube'){ // SonarQube is the name that we configured in manage jenkins > sonarqube 
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=i27-eureka \
                            -Dsonar.host.url=${env.SONAR_URL} \
                            -Dsonar.login=${SONAR_TOKEN} 
                        """
                }
                timeout (time: 2, unit: 'MINUTES') { // NANOSECONDS, SECONDS, MINUTES, HOURS, DAYS
                    script {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        stage ("Docker Build and Push") {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                    }
                }
            }
            // agent {
            //     label 'docker-slave'
            // }
            steps {
                script {
                    dockerBuildandPush().call()
                }

            }
        }
        stage ('Deploy To Dev') {
            when {
                anyOf {
                    expression {
                        params.deployToDev == 'yes'
                    }
                }
            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('dev', '5232', '8232').call()
                    echo "Deployed to Dev Environment Succesfully!!!"
                }
            }
        }
        stage ('Deploy To Test') {
            when {
                anyOf {
                    expression {
                        params.deployToTest == 'yes'
                    }
                }
            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('test', '6232', '8232').call()
                    echo "Deployed to Test Environment Succesfully!!!"
                }
            }
        }
        stage ('Deploy To Stage') {
            when {
                anyOf {
                    expression {
                        params.deployToStage == 'yes'
                    }
                }
            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('stage', '7232', '8232').call()
                    echo "Deployed to Stage Environment Succesfully!!!"
                }
            }
        }
        stage ('Deploy To Prod') {
            when {
                allOf {
                    anyOf {
                        expression {
                            params.deployToProd == 'yes'
                            // other condition as well
                        }
                    }
                    anyOf {
                        branch 'release/*'
                        // one more condition as well
                    }
                }
            }
            steps {
                timeout(time: 300, unit: 'SECONDS') {
                    input message: "Deploying to ${env.APPLICATION_NAME} to production ???", ok: 'yes', submitter: 'mat'
                }
                
                script {
                    imageValidation().call()
                    dockerDeploy('prod', '8232', '8232').call()
                    echo "Deployed to Prod Environment Succesfully!!!"
                }
            }
        }

    }
}

// Create a method to deploy our application into various environments 

def dockerDeploy(envDeploy, hostPort, contPort) {
    return {
        // for every env what will change ?????
        // application name, hostport, container port, container name, environment
        echo "**************************** Deploying to $envDeploy Environment ****************************"
        withCredentials([usernamePassword(credentialsId: 'maha_docker_vm_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
            script {
                // Pull the image on the docker server
                sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
                try {
                    // Stop the container
                    sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker stop ${env.APPLICATION_NAME}-$envDeploy"
                    // Remove the Container
                    sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker rm ${env.APPLICATION_NAME}-$envDeploy"
                } catch(err) {
                    echo "Error Caught: $err"
                }
                // Create the container
                echo "Creating the Container"
                sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker run -d -p $hostPort:$contPort --name ${env.APPLICATION_NAME}-$envDeploy ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
            }
            
        }
    }
}


def imageValidation() {
    return {
        println ("Pulling the Docker image")
        try {
          sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        }
        catch (Exception e) {
            println("OOPS!!!!!, docker image with this tag doesnot exists, So creating the image")
            buildApp().call()
            dockerBuildandPush().call()
        }

    }
}


def buildApp() {
    return {
        echo "Building the ${env.APPLICATION_NAME} Application"
        //mvn command 
        sh 'mvn clean package -DskipTests=true'
        archiveArtifacts artifacts: 'target/*.jar'
    }
}

def dockerBuildandPush() {
    return {
        echo "Starting Docker build stage"
        sh "cp ${WORKSPACE}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd/"
        echo "**************************** Building Docker Image ****************************"
        sh "docker build --force-rm --no-cache --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd"        
        echo "**************************** Login to Docke Repo ****************************"
        sh "docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}"
        echo "**************************** Docker Push ****************************"
        sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
            
    }
}

// ****************************************************** Commented Code ******************************************************
/*
// Deploy to dev flow
1) jenkins should be connecting to the dev server using username and passwrd 
2) The password shoyld be availanle in the credentials

Different environments will have different host ports and same container port(8132)
// for this eureka microservice , i will define the below host ports 
8232
// dev env ====> host port is 5132
// test env ====> host port is 6132
// stage env ====> host port is 7132
// Prod env ======> host port is 8132

*/

