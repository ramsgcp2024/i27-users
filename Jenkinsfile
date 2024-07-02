// This Jenkinsfile is for Eureka microservice
pipeline {
    agent {
        label 'k8s-slave'
    }
    tools {
        maven 'Maven-3.8.8'
        jdk 'JDK-17'
    }
    parameters {
        choice (name: 'buildOnly',
                choices: 'no\nyes',
                description: "Build the Application Only !!!"
                )
        choice (name: 'scanOnly',
                choices: 'no\nyes',
                description: "Scan the Artifact only"
                )
        choice (name: 'dockerPush',
                choices: 'no\nyes',
                description: "Push Image to Docker registry"
                )
        choice (name: 'deployToDev',
                choices: 'no\nyes',
                description: "This will deploy into Dev Environment"
                )
        choice (name: 'deployToTest',
                choices: 'no\nyes',
                description: "This will deploy to test environment"
                )
        choice (name: 'deployToPreprod',
                choices: 'no\nyes',
                description: "This will deploy to Preprod Environment"
                )
        choice (name: 'deployToProd',
                choices: 'no\nyes',
                description: "This will deploy to Prod Environment"
                )
    }
    environment {
        APPLICATION_NAME = "user"
        SONAR_URL = "http://34.125.122.109:9000/"
        SONAR_TOKEN = credentials('sonar_creds')
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        DOCKER_HUB = "docker.io/ramsgcp2024"
        DOCKER_CREDS = credentials('docker_creds')
        //DOCKER_HOST_IP = 0.0.0.0
    }
    stages {
        stage('Build') {
            when {
                anyOf {
                    expression {
                        params.buildOnly == 'yes'
                    }
                }
            }
            steps {
                script {
                    buildApp().call()
                }
            }
        }
        stage('Sonar') {
            when {
                anyOf {
                    expression {
                        params.scanOnly == 'yes'
                    }
                }
            }
            steps {
                echo "Starting sonar scans with Quality Gates"
                //before we go to next step we need to install SonarQube plugin
                // next goto manage Jenkins > System > Add sonarqube > give URL and token for SonarQube 
                withSonarQubeEnv('SonarQube') { //we given 'SonarQube' name at the time of Add sonarqube
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=i27-user \
                        -Dsonar.host.url=${env.SONAR_URL} \
                        -Dsonar.login=${SONAR_TOKEN}
                        """
                }   
              //  timeout (time: 2, unit: 'MINUTES') { //SECONDS, MINUTES, HOURS, DAYS
                //        script {
                  //          waitForQualityGate abortPipeline: true
                    //    }
                }
            }
            stage('Docker Build & Push') {
                when {
                    anyOf {
                        expression {
                            params.dockerPush == 'yes'
                        }
                    }
                }
                steps {
                    script {
                        dockerBuildandPush().call()
                    }
                }
            }

            stage('Deploy to Dev') {
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
                        dockerDeploy('Dev','5232','8232').call()
                        echo "Deployed to Dev Environment Successfully !!!!!"
                    }
                }
            }
            stage('Deploy to Test') {
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
                        dockerDeploy('Test','6232','8232').call()
                        echo "Deployed to Test Environment Successfully !!!!!"
                    }
                }
            }
            stage('Deploy to Preprod') {
                when {
                    anyOf {
                        expression {
                            params.deployToPreprod == 'yes'
                        }
                    }
                }
                steps {
                    script {
                        imageValidation().call()
                        dockerDeploy('Preprod','7232','8232').call()
                        echo "Deployed to Preprod Environment Successfully !!!!!"
                    }
                }
            }
            stage('Deploy to Prod') {
                when {
                    allOf {
                    anyOf {
                        expression {
                            params.deployToProd == 'yes'
                        }
                    }
                     anyOf {
                            branch 'release/*'
                    }
                    }
                }
                steps {
                    timeout(time: 300, unit: 'SECONDS') {
                    input message: "Deploying to ${env.APPLICATION_NAME} to production ???", ok: 'yes', submitter: 'john'
                    }
                    script {
                        imageValidation().call()
                        dockerDeploy('Prod','8232','8232').call()
                        echo "Deployed to Prod Environment Successfully !!!!!"
                    }
                }
            }

        }
        //   
//                     cp ${WORKSPACE}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd/


// docker build --build-args JAR_SOURCE = i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd
       /*
        stage('Docker Format') {
            steps {
                //i27-Eureka-0.0.1-SNAPSHOT.jar
                echo " The Current format is: i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING}"
                // Expected format
                echo "Expected format is: i27-${env.APPLICATION_NAME}-${currentBuild.number}-${BRANCH_NAME}.${env.POM_PACKAGING}" 
            }

        }
        */
    }


def dockerDeploy(envDeploy, hostPort, containerPort) {
    //Application Name, Containar Port, Host Port, container name, environment
    return {
        echo "********************** Deploy to $envDeploy Environment ***********************"
        withCredentials([usernamePassword(credentialsId: 'rama_docker_vm_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) 
        {
        script {
            //Pull the image on the DOCKER Server
            sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        try {
            //stop
            sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker stop ${env.APPLICATION_NAME}-$envDeploy"
            //remove
            sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker rm ${env.APPLICATION_NAME}-$envDeploy"
        }catch(err) {
            echo "Error caught: $err"   
        }
            // create the container
            echo "Creating the container"
            sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker run -d -p $hostPort:$containerPort --name ${env.APPLICATION_NAME}-$envDeploy ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        }
            // some block
            //sh "sshpass -p ${PASSWORD} ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} hostname -i"
            
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
        echo "Starting Docker build and stage"
            sh """
            ls -la
            pwd
            cp ${WORKSPACE}/target/i27-${env.APPLICATION_NAME}-${POM_VERSION}.${POM_PACKAGING} ./.cicd/
            echo "Listing files in .cicd folder"
            ls -la ./.cicd/
            echo "********************** Building DOCKER Image ***********************"
            docker build --force-rm --no-cache --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} .cicd/.
            # docker build -t abc .
            docker images
            echo "********************** DOCKER Login ***********************"
            docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}
            echo "********************** DOCKER Push ***********************"
            docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}
            """
    }
}