// Obtaining an Artifactory server instance defined in Jenkins:

def server = Artifactory.server 'Artifactory Version 4.15.0'

		 //If artifactory is not defined in Jenkins, then create on:
		// def server = Artifactory.newServer url: 'Artifactory url', username: 'username', password: 'password'

//Create Artifactory Maven Build instance
def rtMaven = Artifactory.newMavenBuild()

def buildInfo

pipeline {
    agent any

	tools {
		jdk "Java-1.8"
		maven "Maven-3.5.3"
	}

    stages {
        stage('Clone sources'){
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/anit345/test.git']]])
            }
        }

     	stage('SonarQube analysis') {
	     steps {
		//Prepare SonarQube scanner enviornment
		withSonarQubeEnv('SonarQube6.3') {
		   bat 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.3.0.603:sonar'
		}
	      }
	}

	stage('Artifactory configuration') {

	   steps {
		script {
			rtMaven.tool = 'Maven-3.5.3' //Maven tool name specified in Jenkins configuration

			rtMaven.deployer releaseRepo: 'libs-release-local', snapshotRepo: 'libs-snapshot-local', server: server //Defining where the build artifacts should be deployed to

			rtMaven.resolver releaseRepo:'libs-release', snapshotRepo: 'libs-snapshot', server: server //Defining where Maven Build should download its dependencies from

			rtMaven.deployer.artifactDeploymentPatterns.addExclude("pom.xml") //Exclude artifacts from being deployed

			//rtMaven.deployer.deployArtifacts =false // Disable artifacts deployment during Maven run

			buildInfo = Artifactory.newBuildInfo() //Publishing build-Info to artifactory

			buildInfo.retention maxBuilds: 10, maxDays: 7, deleteBuildArtifacts: true

			buildInfo.env.capture = true
			}
	    }
	}

	stage('Execute Maven') {
		steps {
		   script {

		rtMaven.run pom: 'pom.xml', goals: 'clean install', buildInfo: buildInfo
			}
		}

	}

	stage('Publish build info') {
		steps {
		  script {

		server.publishBuildInfo buildInfo
		}
		}
	}
	    stage('Docker image') {
		steps {
		  script {

		sh """
           cd spring-boot-examples/spring-boot-web-application
           docker build -t Sample_java_docker_image .
        """
		}
		}
	}
	        stage('Deploy') {
		steps {
		  script {

		sh """
        
            gcloud container clusters get-credentials techocamp-1 --region us-central1
            
            #ConfigMaps
            kubectl create configmap spring-config --from-file=spring-boot-examples/spring-boot-web-application/src/main/resources/application.properties
            kubectl describe configmaps spring-config
            
            #Autoscale at 90% CPU usage
            
            kubectl autoscale deployment spring --min=2 --max=5 --cpu-percent=90
            kubectl get hpa
            
            kubectl apply -f spring-boot-examples/spring-boot-web-application/deploy.yml
            kubectl apply -f spring-boot-examples/spring-boot-web-application/service.yml
            
            
        """
		}
		}
	}
}
}
