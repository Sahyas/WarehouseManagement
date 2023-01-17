pipeline {
    agent any

    tools {
        maven 'maven-3.8.6'
        jdk 'jdk-11'
    }

    environment {
        DERBY_START_SERVER_SCRIPT = '/var/lib/jenkins/workspace/apache-derby/bin/startNetworkServer'
        DERBY_SERVER_CHECK = '/var/lib/jenkins/workspace/apache-derby/bin/NetworkServerControl'

        DERBY_RUN_JAR = '/var/lib/jenkins/workspace/apache-derby/lib/derbyrun.jar'

        INIT_DB_SCRIPT = '/var/lib/jenkins/workspace/apache-derby/bin/init_db_script.sql'
        INIT_DB_DATA_SCRIPT = './src/main/resources/initDB.sql'
        CREATE_DB_STRUCTURE_SCRIPT = './src/main/resources/createDB.sql'
        LIST_DATA_SCRIPT = './src/main/resources/listDB.sql'

        EXPECTED_LISTED_DATA = '/var/lib/jenkins/workspace/expected_data'
        ACTUAL_LISTED_DATA = '/var/lib/jenkins/workspace/actual_data'

        ASADMIN = '/var/lib/jenkins/workspace/payara5/bin/asadmin'
        WM_POM_PATH = '/var/lib/jenkins/workspace/WarehouseManagement'
        RESOURCES = '/var/lib/jenkins/workspace/WarehouseManagement/src/main/webapp/WEB-INF/glassfish-resources.xml'

        WAR_PATH = '/var/lib/jenkins/workspace/WarehouseManagement/target/WM-1.1.war'
        APPLICATION_NAME = 'WM-1.1'
    }

    stages {
        stage('Start & verify Apache Derby Server') {
            steps {
                withEnv(['JENKINS_NODE_COOKIE=dontkill']) {
                    sh 'nohup ${DERBY_START_SERVER_SCRIPT} &'
                }
                script {
                    sleep(time:15, unit:"SECONDS")

                    PING = sh (
                        script: '${DERBY_SERVER_CHECK} ping',
                        returnStdout: true
                    ).trim()

                    if (!PING.contains('Uzyskano połączenie dla hosta')) {
                        error(RETURN_MESSAGE)
                    }
                }
            }
        }

        stage('Create DB structure & init data') {
            steps {
                sh 'java -jar ${DERBY_RUN_JAR} ij ${INIT_DB_SCRIPT}'
                sh 'java -jar ${DERBY_RUN_JAR} ij ${CREATE_DB_STRUCTURE_SCRIPT}'
                sh 'java -jar ${DERBY_RUN_JAR} ij ${INIT_DB_DATA_SCRIPT}'

                sh 'java -jar ${DERBY_RUN_JAR} ij ${LIST_DATA_SCRIPT} > ${ACTUAL_LISTED_DATA}'
                script {
                    COMPARISMENT = sh (
                        script: 'cmp --silent ${EXPECTED_LISTED_DATA} ${ACTUAL_LISTED_DATA} || echo "different"',
                        returnStdout: true
                    ).trim()
                    if (COMPARISMENT.contains("different")) {
                        error('Data initialization failed')
                    }
                }
            }
        }

        stage('Start Payara Server') {
            steps {
                script {
                    withEnv(['JENKINS_NODE_COOKIE=dontkill']) {
                        catchError(buildResult: 'SUCCESS') {
                            sh 'nohup ${ASADMIN} start-domain'
                        }
                    }
                }
                sh '${ASADMIN} add-resources ${RESOURCES}'
            }
        }

        stage('Build WAR') {
            steps {
                sh 'mvn -f ${WM_POM_PATH} clean install'
            }
        }

        stage('Deploy Application') {
            steps{
                script {
                    try {
                        sh '${ASADMIN} deploy ${WAR_PATH}'
                    } catch (Exception e) {
                        sh '${ASADMIN} redeploy --name ${APPLICATION_NAME} --properties keepSession=true ${WAR_PATH}'
                    }
                }
            }
        }
    }
}
