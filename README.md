#jenkins-job-examples - Mule Build Pipeline
This project contains example build jobs that can be use in a delivery pipeline for Mule ESB. This includes provision for both continuous integration and continuous delivery. 

### prerequisites
First set up Jenkins with the following plugins
 
* install [Jenkins Build Pipline Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Pipeline+Plugin)
* build/install [Jenkins MMC Plugin](https://github.com/adamsdavis1976/jenkins-mmc-plugin) (todo: submit plugin for Jenkins hosting)
* install [Copy Artifact Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Copy+Artifact+Plugin)
* install [Workspace Cleanup Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Workspace+Cleanup+Plugin)
* install [Extensible Choice Parameter plugin](https://wiki.jenkins-ci.org/display/JENKINS/Extensible+Choice+Parameter+plugin)
* place setting_ci.xml in the `~/.m2` directory and add relevent credentials for CH, MMC etc
* configure 2 mule instances, add them in MMC and label them with servergroups `ci` and `qa` 

### Job configuration 
Clone the jobs in this project and place them in `${JENKINS_HOME}/jobs`

	https://github.com/mulesoft-consulting/jenkins-job-examples

Once the jobs are imported , create a build pipeline with the initial job configured to be `test-project-build`. 

Then Git credentials must be configure in `test-project-build` and `test-project-release-maven`. 

For purposes of derstanding and customisation, additional detail on the examples are below.  

#### Pipeline

##### 1. build step (test-project-build)
This step will normally be triggered by a code check-in and can be regarded as the 'continuous integration' phase.

* Maven Project
* Build Other Projects (Manual) - release step
* Configure git check out youe Mule Project
	* example  `https://github.com/adamsdavis1976/mule-test.git`
* Configure maven goals 
	*  Configure the approaprate maven goal for building and testing your code, this might via  
		* Junit 
		* Munit 
		* External End to end tests 
			* Note:  this [Gist](https://gist.github.com/adamsdavis1976/4bcb4e16f78d837139c8) contains a parent pom that can be used to help deploy (via MMC) and test via SOAPUI. The example mule project about contains this parent pom.   
		* Example maven build phases to execute 
			
					mvn clean deploy 
			
* add `Build other projects (Manual Step)` as a post build actions 
	* Add parameter - `Pass through Git commit` 

##### 2. release step (test-project-release-maven)
This step will be trigger by user input. Jenkins permits execution of jobs to be limited by role but this is not described here. 

* Maven Project 
* Configure Git plugin 
	* example `https://github.com/adamsdavis1976/mule-test.git`
	* Branches to build
		* `${GIT_COMMIT}`
	* additional behaviors 
		* check out to specific local branch 
			* `"master"`
* Add Build step - Goals 
	
		clean release:clean release:prepare -Dskip.deploy.nexus=true`
	
* Post Steps
	* Add the following shell script

			suffix="-SNAPSHOT"
			RELEASE_POM_VERSION_EDITED=${POM_VERSION%$suffix}
			echo RELEASE_POM_VERSION=$RELEASE_POM_VERSION_EDITED > version.properties 

	* Invoke top level maven target 
			
			release:perform
	
	* Archive the artifact
		* File to archive 
				`**/target/*.zip`
			
	* Build other Projects (manual step) 
		* nominate the deploy phase of choice
			* Predefined parameters
		
					ARTEFACT_BUILD_NUMBER=$BUILD_NUMBER
					POM_ARTIFACTID=$POM_ARTIFACTID
					
			* Parameters from properties file 
			
					version.properties

Note: I required a more recent version of the maven release plugin to get the git commits correct. This is included in this [parent pom](https://gist.github.com/adamsdavis1976/4bcb4e16f78d837139c8).

##### 3. deploy to mule esb step (test-project-deploy-qa-esb)
Again, this step will be trigger by user input. Jenkins permits execution of jobs to be limited by role but this is not described here. This step can be reqgarded as the 'continuous deployment' phase. 

Separate instance of this step can be created for diferent environments. For example 

>QA --> UAT --> Production

* Freestyle Project
* Delete workspace before build starts
* Copy artfacts from another project
	* specific build `$ARTEFACT_BUILD_NUMBER`
	* artifacts to copy `**/target/*.zip`
	
* Deploy to Mule Management console 
	* Artifact Version 
		`$RELEASE_POM_VERSION`
	* Artifact Name
		`$POM_ARTIFACTID`
	* File Location
		`**/target/*.zip`


##### 4. deploy to cloudhub step (test-project-deploy-qa-cloudhub)
Again, this step will be trigger by user input. Jenkins permits execution of jobs to be limited by role but this is not described here. This step can be reqgarded as the 'continuous deployment' phase. 

As above, separate instance of this step can be created for diferent environments. 

* Freestyle Project
* Delete workspace before build starts
* Copy artfacts from another project
	* specific build `$ARTEFACT_BUILD_NUMBER`
	* artifacts to copy `**/target/*.zip`
* Deploy to Cloudhub via `org.mule.tools.maven:cloudhub-maven-plugin` called as a standalone command

		mvn org.mule.tools.maven:cloudhub-maven-plugin:undeploy -Dcloudhub.username=username -Dcloudhub.password=password -Dcloudhub.domain=testdomain 
		mvn org.mule.tools.maven:cloudhub-maven-plugin:deploy -Dcloudhub.username=username -Dcloudhub.password=password -Dcloudhub.domain=testdomain -Dapplication=application.zip -Dcloudhub.muleVersion=3.6.0
		
* Note1: `cloudhub-maven-plugin` is available in maven central so no special configuration should be required for settings.xml.
* Note2: `cloudhub.username` and `cloudhub.password` can be placed in settings.xml and [encrypted](https://maven.apache.org/guides/mini/guide-encryption.html) if necessary.

#### Standalone Deploy

##### 1. Standalone Deploy from a Nexus Instance (deploy-test-app)
This Job allows you to select a specific version of an Artefact from a Nexus repository and Deploy it to an MMC instance. This could be extended to provide an environment drop down so as to allow an artefact to be deployed to any environment.

