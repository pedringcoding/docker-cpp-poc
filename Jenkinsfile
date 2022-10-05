#!groovy
pipeline {

	options {
        // Retain history on the last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Add job at buffer
        disableConcurrentBuilds()
        // Set a timeout on the total execution time of the job
        timeout(time: 10, unit: 'MINUTES')
    }

    triggers {
        // This will poll SCM for any changes every five minutes. 
        // If a change is detected since the last build it job will be triggered to build the changes.
        pollSCM 'H/5 * * * 1-5'
    }

    agent any

    parameters {
        string(name: 'secret_id', defaultValue: 'generic-user', description: 'The basic-auth identifier from jenkins credentials')
    }

    environment {
        // Verifier stage when error occurs during step execution
        LAST_STAGE_NAME = ""
        // secrets
        AUTH_CREEDS = credentials("${params.secret_id}")

        // to distribution component
        app_name = ""
        app_version = ""

        // Docker Trusted Registry to distribute docker images
        docker_projectName = "simulation"
        docker_defaultImageName = ""
        docker_releaseImageName = ""
    }

	stages {
        stage('ENVIRONMENT PREPARATION') {
            steps {
                script {
                    app_name = sh (returnStdout: true, script: "grep -oP '(?<=<name>).*?(?=</name>)' package.xml | tr -d '\\n'")
                    app_version = sh (returnStdout: true, script: "grep -oP '(?<=<version>).*?(?=</version>)' package.xml | tr -d '\\n'")
                    echo "Application is ${app_name}:${app_version}"

                    docker_defaultImageName = "${docker_projectName}/${app_name}:latest"
                    docker_releaseImageName = "${docker_projectName}/${app_name}:${app_version}"
                    echo "Docker tags will be are ${docker_defaultImageName} and ${docker_releaseImageName}"
                }
            }
            post {
                failure {
                    script {
                        LAST_STAGE_NAME = env.STAGE_NAME
                    }
                }
            }
        }

        stage('BUILD') {
            parallel {
                stage('Clang') {
                    environment {
                        DOCKERFILE = 'Dockerfile.clang'
                        TAG_DEFAULT = "${docker_defaultImageName}"
                        TAG_RELEASE = "${docker_releaseImageName}"
                    }
                    steps {
                        sh "docker build -t ${TAG_DEFAULT} -f ${DOCKERFILE} ."
                        script {
                            if (env.BRANCH_NAME == 'main') {
                                sh "docker tag ${TAG_DEFAULT} ${TAG_RELEASE}"
                            }
                        }
                    }
                    post {
                        failure {
                            script {
                                LAST_STAGE_NAME = env.STAGE_NAME
                            }
                        }
                    }
                }
                stage('GCC') {
                    environment {
                        DOCKERFILE = 'Dockerfile.gcc'
                        TAG_DEFAULT = "${docker_defaultImageName}"
                        TAG_RELEASE = "${docker_releaseImageName}"
                    }
                    steps {
                        sh "docker build -t ${TAG_DEFAULT} -f ${DOCKERFILE} ."
                        script {
                            if (env.BRANCH_NAME == 'main') {
                                sh "docker tag ${TAG_DEFAULT} ${TAG_RELEASE}"
                            }
                        }
                    }
                    post {
                        failure {
                            script {
                                LAST_STAGE_NAME = env.STAGE_NAME
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                if (currentBuild.currentResult != 'SUCCESS' && (env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'develop')) {
                    echo "Please, check the ${currentBuild.currentResult} status when executing ${LAST_STAGE_NAME} step. Pipeline duration: ${currentBuild.durationString}."
                }
            }
        }
    }
}