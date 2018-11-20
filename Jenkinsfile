import groovy.json.JsonSlurper
import groovy.json.JsonOutput
pipeline {
    agent any
    environment {
        mvnHome = ''
        artiServer = ''
        rtMaven = ''
        buildInfo = ''
    }
    stages {
        //artifactory
        def artiServer = Artifactory.server 'art'
        def rtMaven = Artifactory.newMavenBuild()
        def buildInfo = Artifactory.newBuildInfo()
        def artifactoryUrl = 'http://139.199.24.190:8081/artifactory/'
        def resolveSnapshotRepo = 'lib-snapshot'
        def resolveReleaseRepo = 'lib-release'
        def deploySnapshotRepo = 'lib-snapshot'
        def deployReleaseRepo = 'lib-release'

        def promotionSourceRepo = 'lib-release-local'
        def promotionTargetRepo = 'lib-snapshot-local'

        //maven
        def mavenTool = 'maven'
        def mavenGoals = 'clean install'
        def pomPath = 'maven-example/pom.xml'

        //git
        def gitUrl = 'https://github.com/JFrogChina/project-example.git'
        def branch = 'master'
        def gitCredentialsId = 'my-git-hub'

        //sonar
        def sonarUrl = 'http://192.168.199.201:9000'
        def sonarServer = 'sonar'
        def sonarScannerTool = 'sonarClient'
        def sonarProjectKey = 'mvn-e2e-pipeline-demo'
        def sonarSources = 'maven-example/multi3/src'



        stage('env capture') {
            buildInfo.env.capture = true
        }
        stage('SCM') {
            git(url: gitUrl, branch: branch, changelog: true, credentialsId: gitCredentialsId, poll: true)
        }
        //环境配置
        stage('Prepare') {
            rtMaven.resolver server: artiServer, releaseRepo: resolveReleaseRepo, snapshotRepo: resolveSnapshotRepo
            rtMaven.deployer server: artiServer, releaseRepo: deployReleaseRepo, snapshotRepo: deploySnapshotRepo
            rtMaven.tool = mavenTool
            rtMaven.run pom: pomPath, goals: mavenGoals, buildInfo: buildInfo
            rtMaven.deployer.deployArtifacts = false

        }
        //Sonar 静态代码扫描
        stage('Sonar') {
            // Sonar scan
            def scannerHome = tool sonarScannerTool;
            withSonarQubeEnv(sonarServer) {
                sh "${scannerHome}/bin/sonar-runner -Dsonar.projectKey=" + sonarProjectKey + " -Dsonar.sources=" + sonarSources
            }
        }
        //添加sonar扫描结果到包上
        stage("Sonar Quality Gate") {

            timeout(time: 1, unit: 'HOURS') {
                // Just in case something goes wrong, pipeline will be killed after a timeout
                def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                } else {
                    //获取sonar扫描结果
                    def getSonarIssuesCmd = "curl  GET -v " + sonarUrl + "/api/issues/search?componentRoots=${JOB_NAME}";
                    process = ['bash', '-c', getSonarIssuesCmd].execute().text

                    //增加sonar扫描结果到artifactory
                    rtMaven.deployer.addProperty("qulity.gate.sonarUrl", sonarUrl + "/dashboard/index/${JOB_NAME}").addProperty("qulity.gate.sonarIssue", issueMap.total)
                }
            }
        }
        stage('add jiraResult') {
            def requirements = getRequirementsIds();
            echo "requirements : ${requirements}"
            //def revisionIds = getRevisionIds();
            def revisionIds = "";
            echo "revisionIds : ${revisionIds}"
            rtMaven.deployer.addProperty("project.issues", requirements).addProperty("project.revisionIds", revisionIds)
        }
        //maven 构建
        stage('mvn build') {
            rtMaven.deployer.deployArtifacts buildInfo
            artiServer.publishBuildInfo buildInfo
        }
        //进行测试
        stage('basic test') {
            echo "add test step"
        }

        stage('xray scan') {
            def xrayConfig = [
                    'buildName'  : env.JOB_NAME,
                    'buildNumber': env.BUILD_NUMBER,
                    'failBuild'  : false
            ]
            def xrayResults = artiServer.xrayScan xrayConfig
            echo xrayResults as String
        }

        //promotion操作，进行包的升级
        stage('promotion') {
            def promotionConfig = [
                    'buildName'          : buildInfo.name,
                    'buildNumber'        : buildInfo.number,
                    'targetRepo'         : promotionTargetRepo,
                    'comment'            : 'this is the promotion comment',
                    'sourceRepo'         : promotionSourceRepo,
                    'status'             : 'Released',
                    'includeDependencies': false,
                    'failFast'           : true,
                    'copy'               : true
            ]
            artiServer.promote promotionConfig
        }
        //进行部署
        stage('deploy') {
            // def deployCmd = "ansible 127.0.0.1 -m get_url -a 'url=" + artifactoryUrl + promotionTargetRepo + "/org/jfrog/test/multi3/7.0.1-SNAPSHOT/multi3-" + warVersion + ".war  dest=/home/tomcat/apache-tomcat-8.5.33/webapps/demo.war force=true url_username=admin url_password=AKCp5bBXY7TmCE3s13XrGRM3ejcCGfiNgmtAFbaR8yNNLuZW3f7vFhzbv9NxBLP1AWxJr9Zj2'"
            // //sh "ansible 10.1.1.1 -m shell -a '/tomcat/startup.sh' "
            // echo deployCmd
            // process = ['bash', '-c', deployCmd].execute().text

            // commandText = "curl  -X PUT \""+ artifactoryUrl + "api/storage/libs-release-local/org/jfrog/test/multi3/7.0.1-SNAPSHOT/multi3-" + warVersion + ".war?properties=deploy.tool=ansible;deploy.env=127.0.0.1\" -uadmin:AKCp5bBXY7TmCE3s13XrGRM3ejcCGfiNgmtAFbaR8yNNLuZW3f7vFhzbv9NxBLP1AWxJr9Zj2";
            // echo commandText
            // ['bash', '-c', commandText].execute().text

            // def stopCmd = "ansible 127.0.0.1 -m shell -a '/home/tomcat/apache-tomcat-8.5.33/bin/shutdown.sh'"
            // echo stopCmd
            // ['bash', '-c', stopCmd].execute().text
            // def startCmd = "ansible 127.0.0.1 -m shell -a '/home/tomcat/apache-tomcat-8.5.33/bin/startup.sh'"
            // echo startCmd
            // ['bash', '-c', startCmd].execute().text
        }
    }

	// @NonCPS
        // def getRequirementsIds() {
        //     def reqIds = "";
        //     final changeSets = currentBuild.changeSets
        //     echo 'changeset count:' + changeSets.size().toString()
        //     final changeSetIterator = changeSets.iterator()
        //     while (changeSetIterator.hasNext()) {
        //         final changeSet = changeSetIterator.next();
        //         def logEntryIterator = changeSet.iterator();
        //         while (logEntryIterator.hasNext()) {
        //             final logEntry = logEntryIterator.next()
        //             def patten = ~/#[\w\-_\d]+/;
        //             def matcher = (logEntry.getMsg() =~ patten);
        //             def count = matcher.getCount();
        //             for (int i = 0; i < count; i++) {
        //                 reqIds += matcher[i].replace('#', '') + ","
        //             }
        //         }
        //     }
        //     return reqIds;
        // }
        // // @NonCPS
        // def getRevisionIds() {
        //     def reqIds = "";
        //     final changeSets = currentBuild.changeSets
        //     final changeSetIterator = changeSets.iterator()
        //     while (changeSetIterator.hasNext()) {
        //         final changeSet = changeSetIterator.next();
        //         def logEntryIterator = changeSet.iterator();
        //         while (logEntryIterator.hasNext()) {
        //             final logEntry = logEntryIterator.next()
        //             reqIds += logEntry.getRevision() + ","
        //         }
        //     }
        //     return reqIds
        // }


}