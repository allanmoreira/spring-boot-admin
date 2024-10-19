pipeline {
    agent none
    environment {
        POM_XML_VERSION = ""
        BRANCH_DEFAULT = "master"
        POM_XML_FILE = "pom.xml"
        ARTIFACT_ID = ""
        ARTIFACT_NAME = ""
        JAR_NAME = ""
        BUILD_PATH = '/home/jenkins/builds'
        DOCKERFILE_PATH = "Dockerfile"
        CONTEXT_BUILD = "${BUILD_PATH}"
        PATH_ARTIFACT = ""
    }
    stages {
        stage ('Build') {
            agent any
            steps {
                script {
                    echo 'Read pom file'
                    pom = readMavenPom file: "$POM_XML_FILE"

                    POM_XML_VERSION = "${pom.version}"
                    ARTIFACT_ID = "${pom.artifactId}"
                    ARTIFACT_NAME = "${ARTIFACT_ID}-${POM_XML_VERSION}"
                    JAR_NAME = "${ARTIFACT_NAME}.jar"
                    PATH_ARTIFACT = "${BUILD_PATH}/${ARTIFACT_ID}/${JAR_NAME}"

                    echo "Run maven package to ${POM_XML_VERSION}"
                    sh 'mvn clean package -Dmaven.test.skip=true'

                    sh 'mkdir /home/jenkins/builds  || true'

                    sh "mkdir /home/jenkins/builds/${ARTIFACT_ID}  || true"

                    echo "Copy ${ARTIFACT_NAME} to ${BUILD_PATH}"
                    sh "cp target/${JAR_NAME} ${BUILD_PATH}/${ARTIFACT_ID}"
                }
            }
        }
        stage ('Release Candidate') {
            agent any
            environment {
                GITHUB_CREDENTIALS = credentials('github')
            }
            steps {
                script {
                    sh "git checkout -b ${BRANCH_DEFAULT} remotes/origin/${BRANCH_DEFAULT}"

                    sh "git config remote.origin.url https://'${GITHUB_CREDENTIALS_USR}:${GITHUB_CREDENTIALS_PSW}'@github.com/${GITHUB_CREDENTIALS_USR}/${ARTIFACT_ID}.git"

                    echo 'Read pom file'
                    pom = readMavenPom file: "$POM_XML_FILE"
                    def version = pom.version.toString().split("\\.")
                    if(params.NOVA_VERSAO == true){
                        version[0] = version[0].toInteger()+1
                        version[1] = 0
                        version[2] = 0
                    } else {
                        version[2] = version[2].toInteger()+1
                    }
                    pom.version = version.join('.')
                    writeMavenPom model: pom, file: "${POM_XML_FILE}", name: 'Write Maven POM file'
                    POM_XML_VERSION = "${pom.version}"
                    echo "Incremented POM project version to ${POM_XML_VERSION}"

                    sh "git tag -a '${POM_XML_VERSION}' -m \"tag ${POM_XML_VERSION} gerada\""

                    sh "git push origin '${pom.version}'"

                    sh "git add ${POM_XML_FILE}"

                    sh 'git commit -m "POM version increased"'

                    sh "git push origin ${BRANCH_DEFAULT}"
                }
            }
        }
        stage ('Deploy') {
            agent any
            options {
                skipDefaultCheckout true
            }
            steps {
                script {

                    DOCKER_IMAGE = "${ARTIFACT_ID}-image"
                    DOCKER_CONTAINER = "${ARTIFACT_ID}-container"

                    sh "docker stop ${DOCKER_CONTAINER} || true"

                    sh "docker rm ${DOCKER_CONTAINER} || true"

                    sh "docker rmi ${DOCKER_IMAGE} || true"

                    sh "docker build -t ${DOCKER_IMAGE} -f ${DOCKERFILE_PATH} ${CONTEXT_BUILD} --build-arg JAR_FILE=${ARTIFACT_ID}/${JAR_NAME}"

                    sh "docker run --rm --network=host -d --name ${DOCKER_CONTAINER} ${DOCKER_IMAGE}"
                }
            }
        }
        /*
        stage ('Archive') {
            agent any
            environment {
                ARTIFACTORY_CREDENTIALS = credentials('artifactory')
            }
            steps {
                script {
                    CURL = 'curl -u ${ARTIFACTORY_CREDENTIALS_USR}:${ARTIFACTORY_CREDENTIALS_PSW} -X PUT $URL_ARTIFACTORY'

                    sh name: "Upload",
                    script: "$CURL/$ARTIFACT_ID/$JAR_NAME -T $PATH_ARTIFACT"

                    sh name: "Remove",
                    script:  "rm $PATH_ARTIFACT"
                }
            }
        }
        */
    }
}