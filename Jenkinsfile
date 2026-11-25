pipeline {
    agent none
    environment {
        POM_XML_VERSION = ""
        BRANCH_DEFAULT = "master"
        POM_XML_FILE = "pom.xml"
        ARTIFACT_ID = ""
        ARTIFACT_NAME = ""
        JAR_NAME = ""
        BUILD_PATH      = "/home/usr_pub/builds"
        DOCKERFILE_PATH = "Dockerfile"
        CONTEXT_BUILD   = "${BUILD_PATH}"
        PATH_ARTIFACT = ""
        SPRING_BOOT_ADMIN_UI_PUBLIC_URL = "http://moreirallan.com/spring-boot-admin"
    }
    stages {
        stage ('Build') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-21'
                    args '-v /home/usr_pub/jenkins-data/jenkins_home/.m2:/var/maven/.m2 -e MAVEN_CONFIG=/var/maven/.m2'
                    reuseNode true
                }
            }
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

                    sh "mkdir ${BUILD_PATH}  || true"

                    sh "mkdir ${BUILD_PATH}/${ARTIFACT_ID}  || true"

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
                    // sh "git checkout -b ${BRANCH_DEFAULT} remotes/origin/${BRANCH_DEFAULT}"

                    // Identidade do autor dos commits (o container do Jenkins não tem uma configurada)
                    sh 'git config user.name "Jenkins"'
                    sh 'git config user.email "jenkins@localhost"'

                    sh "git config remote.origin.url https://'${GITHUB_CREDENTIALS_USR}:${GITHUB_CREDENTIALS_PSW}'@github.com/${GITHUB_CREDENTIALS_USR}/${ARTIFACT_ID}.git"

                    sh "git pull --rebase origin ${BRANCH_DEFAULT}"

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

                    sh "git add ${POM_XML_FILE}"

                    sh 'git commit -m "POM version increased"'

                    // Tag criada após o commit, para apontar para o pom com a nova versão
                    sh "git tag -a '${POM_XML_VERSION}' -m \"tag ${POM_XML_VERSION} gerada\""

                    // O Jenkins faz checkout em detached HEAD: é preciso enviar o HEAD atual
                    // para a branch remota, e não a branch local (desatualizada)
                    sh "git push origin HEAD:${BRANCH_DEFAULT}"

                    sh "git push origin '${POM_XML_VERSION}'"
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

                    sh "docker run --rm -d \
                          --name ${DOCKER_CONTAINER} \
                          -p 127.0.0.1:8081:8081 \
                          -e SERVER_FORWARD_HEADERS_STRATEGY=framework \
                          -e SPRING_BOOT_ADMIN_UI_PUBLIC_URL=${SPRING_BOOT_ADMIN_UI_PUBLIC_URL}/ \
                          ${DOCKER_IMAGE}"
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