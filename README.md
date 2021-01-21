# Openshift CI/CD for simple Java SpringBoot App with Gradle

This Project Handle the basic CI/CD of Java application using Gradle   

To run locally with Gradle installed:   

```
gradle tasks // to get the list of available tasks
gradle test // to run the unit tests
gradle build // to build the application
gradle bootjar //to build the jar file of this SpringBoot app

```

To use Jenkins on Openshift for CI/CD, first we need to build Gradle Jenkins Slave template to use in our CI/CD 

## 1) Build The Environment

You can run use the build :  

```
oc project cicd //this is the project for cicd

oc create -f bc_jenkins_slave_template.yaml -n cicd //this will add the template to use 
or you can use it directly from the GitHub: oc process -f https://raw.githubusercontent.com/osa-ora/simple_java_gradle/main/cicd/bc_jenkins_slave_template.yaml -n cicd | oc create -f -

Now use the template to create the Jenkins slave template
oc describe template jenkins-slave-template //to see the template details
oc process -p GIT_URL=https://github.com/osa-ora/simple_java_gradle -p GIT_BRANCH=main -p GIT_CONTEXT_DIR=cicd -p DOCKERFILE_PATH=dockerfile_gradle -p IMAGE_NAME=gradle-jenkins-slave jenkins-slave-template  | oc create -f -

oc start-build gradle-jenkins-slave 
oc logs bc/gradle-jenkins-slave -f

oc new-app jenkins-persistent  -p MEMORY_LIMIT=2Gi  -p VOLUME_CAPACITY=4Gi -n cicd

oc project dev //this is project for application development
oc policy add-role-to-user edit system:serviceaccount:cicd:default -n dev
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n dev

```


## 2) Configure Jenkins 
In case you completed 1st step before provision Openshift Jenkins, it will auto-detect the slave gradle image based on the label and annotation and no thing need to be done to configure it, otherwise you can do it manually for existing Jenkins installation

From inside Jenkins --> go to Manage Jenkins ==> Configure Jenkins then scroll to cloud section:
https://{JENKINS_URL}/configureClouds
Now click on Pod Templates, add new one with name "gradle-jenkins-slave", label "gradle-jenkins-slave", container template name "jnlp", docker image "image-registry.openshift-image-registry.svc:5000/cicd/gradle-jenkins-slave" 

See the picture:
<img width="1258" alt="Screen Shot 2021-01-04 at 12 42 20" src="https://user-images.githubusercontent.com/18471537/103526980-5432c980-4e8a-11eb-8bb3-c51067b8fa95.png">

## 3) (Optional) SonarQube on Openshift
Provision SonarQube for code scanning on Openshift using the attached template.
oc process -f https://raw.githubusercontent.com/osa-ora/simple_java_gradle/main/cicd/sonarqube-persistent-template.yaml | oc create -f -

Open SonarQube and create new project, give it a name, generate a token and use them as parameters in our next CI/CD steps

<img width="945" alt="Screen Shot 2021-01-03 at 16 07 57" src="https://user-images.githubusercontent.com/18471537/103480605-e630c880-4ddd-11eb-91de-c43f2882da84.png">

Make sure to select Gradle here.

## 4) Build Jenkins CI/CD using Jenkins File

Now create new pipeline for the project, where we checkout the code, run unit testing, run sonar qube analysis, build the application, get manual approval for deployment and finally deploy it on Openshift.
Here is the content of the file (as in the cicd\jenkinsfile):

```
// Maintaned by Osama Oransa
// First execution will fail as parameters won't populated
// Subsequent runs will succeed if you provide correct parameters
pipeline {
	options {
		// set a timeout of 20 minutes for this pipeline
		timeout(time: 20, unit: 'MINUTES')
	}
    agent {
    // Using the gradle builder agent
       label "gradle-jenkins-slave"
    }
  stages {
    stage('Setup Parameters') {
            steps {
                script { 
                    properties([
                        parameters([
                        choice(
                                choices: ['No', 'Yes'], 
                                name: 'firstDeployment',
                                description: 'First Deployment?'
                            ),
                        choice(
                                choices: ['Yes', 'No'], 
                                name: 'runSonarQube',
                                description: 'Run Sonar Qube Analysis?'
                            ),
                        string(
                                defaultValue: 'dev', 
                                name: 'proj_name', 
                                trim: true,
                                description: 'Openshift Project Name'
                            ),
                        string(
                                defaultValue: 'gradle-app', 
                                name: 'app_name', 
                                trim: true,
                                description: 'Gradle Application Name'
                            ),
                        string(
                                defaultValue: 'https://github.com/osa-ora/simple_java_gradle', 
                                name: 'git_url', 
                                trim: true,
                                description: 'Git Repository Location'
                            ),
                        string(
                                defaultValue: 'http://sonarqube-cicd.apps.cluster-894c.894c.sandbox1092.opentlc.com', 
                                name: 'sonarqube_url', 
                                trim: true,
                                description: 'Sonar Qube URL'
                            ),
                        string(
                                defaultValue: 'gradle', 
                                name: 'sonarqube_proj', 
                                trim: true,
                                description: 'Sonar Qube Project Name'
                            ),
                        password(
                                defaultValue: '744a7a17b4d848d45f16575a6f4b0c754cda6fc5', 
                                name: 'sonarqube_token', 
                                description: 'Sonar Qube Token'
                            )
                        ])
                    ])
                }
            }
    }
    stage('Code Checkout') {
      steps {
        git branch: 'main', url: '${git_url}'
        sh "ls -l"
      }
    }
    stage('Unit Testing & Code Coverage') {
      steps {
        sh "gradle clean build test"
        sh "gradle jacocoTestReport"
        sh "gradle jacocoTestCoverageVerification"
        archiveArtifacts '**/TEST-*.xml'
        archiveArtifacts 'build/jacoco/test/**/*.*'
      }
    }
    stage('Code Scanning by Sonar Qube') {
        when {
            expression { runSonarQube == "Yes" }
        }
        steps {
            sh "gradle sonarqube -Dsonar.login=${sonarqube_token} -Dsonar.host.url=${sonarqube_url} -Dsonar.projectKey=${sonarqube_proj}"
        }
    }
    stage('Build Deployment Package'){
        steps{
            sh "gradle bootjar"
            archiveArtifacts 'build/libs/**/*.*'
        }
    }
    stage('Deployment Approval') {
        steps {
            timeout(time: 5, unit: 'MINUTES') {
                input message: 'Proceed with Application Deployment?', ok: 'Approve Deployment'
            }
        }
    }
    stage('Initial Deploy To Openshift') {
        when {
            expression { firstDeployment == "Yes" }
        }
        steps {
            sh "oc project ${proj_name}"
            sh "oc new-build --image-stream=java:latest --binary=true --name=${app_name}"
            sh "oc start-build ${app_name} --from-dir=build/libs/."
            sh "oc logs -f bc/${app_name}"
            sh "oc new-app ${app_name} --as-deployment-config"
            sh "oc expose svc ${app_name} --port=8080 --name=${app_name}"
        }
    }
    stage('Incremental Deploy To Openshift') {
        when {
            expression { firstDeployment == "No" }
        }
        steps {
            sh "oc project ${proj_name}"
            sh "oc start-build ${app_name} --from-dir=build/libs/."
            sh "oc logs -f bc/${app_name}"
        }
    }
  }
} // pipeline
```
As you can see this pipeline pick the gradle slave image that we built, note the label must match what we configurd before:
```
agent {
    // Using the gradle builder agent
       label "gradle-jenkins-slave"
    }
```

The pipeline uses many parameters and in first execution it failed, then in subsequent execution the parameters will be available

<img width="1294" alt="Screen Shot 2021-01-20 at 22 15 00" src="https://user-images.githubusercontent.com/18471537/105334042-17cbd100-5bdf-11eb-9066-a08691cc66c7.png">

## 5) Deployment Across Environments

Environments can be another Openshift project in the same Openshift cluster or in anither cluster.

In order to do this for the same cluster, you can enrich the pipeline and add approvals to deploy to a new project, all you need is to have the required privilages using "oc policy" as we did before and add deploy stage in the pipeline script to deploy into this project.

```
oc project {project_name} //this is new project to use
oc policy add-role-to-user edit system:serviceaccount:cicd:default -n {project_name}
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n {project_name}
```
Add more stages to the pipleine scripts like:
```
    stage('Deployment to Staging Approval') {
        steps {
            timeout(time: 5, unit: 'MINUTES') {
                input message: 'Proceed with Application Deployment in Staging environment ?', ok: 'Approve Deployment'
            }
        }
    }
    stage('Deploy To Openshift Staging') {
      steps {
        sh "oc project ${staging_proj_name}"
        sh "oc start-build ${app_name} --from-dir=."
        sh "oc logs -f bc/${app_name}"
      }
    }
```
You can use oc login command with different cluster to deploy the application into different clusters. 
Also you can use Openshift plugin and configure different Openshift cluster to automated the deployments across many environments:

```
stage('preamble') {
	steps {
		script {
			openshift.withCluster() {
			//could be openshift.withCluster( 'another-cluster' ) {
				//name references a Jenkins cluster configuration
				openshift.withProject() { 
				//coulld be openshift.withProject( 'another-project' ) {
					//name references a project inside the cluster
					echo "Using project: ${openshift.project()} in cluster:  ${openshift.cluster()}"
				}
			}
		}
	}
}
```
And then configure any additonal cluster (other than the default one which running Jenkins) in Openshift Client plugin configuration section:

<img width="1400" alt="Screen Shot 2021-01-05 at 10 24 45" src="https://user-images.githubusercontent.com/18471537/103623100-4b043400-4f40-11eb-90cf-2209e9c4bde2.png">



