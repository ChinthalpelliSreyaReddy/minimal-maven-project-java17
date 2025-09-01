def registry ="https://sreyajfrog.jfrog.io/"
pipeline {
    agent any
	
tools{
   jdk 'jdk17'
   maven 'maven3'
}

environment{
   SCANNER_ENV = tool 'sonar_scanner'
}
   
        stages{
		stage('build') {
            steps {
                echo '------- build started----------'
				sh 'mvn clean install -DskipTests=true'
				echo '------- build completed----------'
            }
        }
		
		stage('test') {
            steps {
                echo '------- testing started----------'
				sh 'mvn surefire-report:report'
				echo '------- testing completed----------'
            }
        }
		
        stage('sonarqube analysis') {
            steps {
                withSonarQubeEnv('sonar_server') {
                sh ''' 
	          	${SCANNER_ENV}/bin/sonar-scanner \
	         	-Dsonar.projectName=jenkins-website \
		        -Dsonar.projectKey=jenkins-website \
	            -Dsonar.java.binaries=target
		        '''
               }
            }
        }
		
		stage('Quality Gate') {
    steps {
        script {
            timeout(time: 1, unit: 'HOURS') {
                def qg = waitForQualityGate(abortPipeline: false)
                if (qg.status != "OK") {
                    echo "⚠️ Warning: Quality Gate failed but continuing pipeline. Status = ${qg.status}"
                } else {
                    echo "✅ Quality Gate passed"
                }
            }
        }
    }
}

stage("Jar Publish") {
    steps {
        script {
            echo '<--------------- Jar Publish Started ---------------->'

            // Connect to Artifactory
            def server = Artifactory.newServer(
                url: "${registry}/artifactory",   // Example: http://<jfrog-host>:8082/artifactory
                credentialsId: "jfrog_cred"    // Jenkins credentials ID for JFrog
            )

            // Custom properties for tracking build
            def properties = "buildId=${env.BUILD_ID},commitId=${GIT_COMMIT}"

            // Upload spec (source pattern → target repo path)
            def uploadSpec = """{
                "files": [
                    {
                        "pattern": "target/*.jar",
                        "target": "sreya_jenkins-libs-release-local/{1}",
                        "flat": "false",
                        "props": "${properties}",
                        "exclusions": ["*.sha1", "*.md5"]
                    }
                ]
            }"""

            // Upload JARs to JFrog
            def buildInfo = server.upload(uploadSpec)

            // Collect environment vars
            buildInfo.env.collect()

            // Publish build info to JFrog
            server.publishBuildInfo(buildInfo)

            echo '<--------------- Jar Publish Completed ---------------->'
        }
    }
}

		   
		stage('trivy') {
            steps {
               sh 'trivy fs . --format table -o trivy-report.txt'
            }
        }
    }
	post {
        always {
            cleanWs() // Clean workspace after build
        }
    }
}

