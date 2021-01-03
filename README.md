# Openshift CI/CD for simple Java SpringBoot App with Gradle Build

This Project Handle the basic CI/CD of Java application using Gradle   

To run locally with Gradle installed:   

```
gradle tasks // to get the list of available tasks
gradle test // to run the unit tests
gradle build // to build the application
gradle buildjar //to build the jar file of this SpringBoot app

```

To use Jenkins on Openshift for CI/CD, first we need to build Gradle Jenkins Slave template to use in our CI/CD 

## 1) Build The Environment

You can run use the build :  

```
oc project cicd //this is the project for cicd
oc project dev //this is project for application development

oc new-app jenkins-persistent  -n cicd
oc policy add-role-to-user edit system:serviceaccount:cicd:default -n dev
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n dev

oc create -f bc_jenkins_slave.yaml -n cicd //this will add the template to use 
or you can use it directly from the GitHub: oc process -f https://raw.githubusercontent.com/osa-ora/simple_java_gradle/main/cicd/bc_jenkins_slave.yaml -n cicd | oc create -f -

Now use the template to create the Jenkins slave template
oc describe template jenkins-slave //to see the template details
oc process -p GIT_URL=https://github.com/osa-ora/simple_java_gradle -p GIT_BRANCH=main -p GIT_CONTEXT_DIR=cicd -p DOCKERFILE_PATH=dockerfile_gradle -p IMAGE_NAME=gradle-jenkins-slave jenkins-slave  | oc create -f -

oc start-build gradle-jenkins-slave 
oc logs bc/gradle-jenkins-slave -f
```


## 2) Configure Jenkins 
From inside Jenkins --> go to Manage Jenkins ==> Configure Jenkins then scroll to cloud section:
https://{JENKINS_URL}/configureClouds
Now click on Pod Templates, add new one with name "gradle", label "gradle", container template name "jnlp", docker image "image-registry.openshift-image-registry.svc:5000/cicd/gradle-jenkins-slave" 

See the picture:
<img width="1325" alt="Screen Shot 2021-01-03 at 15 36 38" src="https://user-images.githubusercontent.com/18471537/103480527-8d613000-4ddd-11eb-84b4-3d589a192ef3.png">

## 3) (Optional) SonarQube on Openshift
Provision SonarQube for code scanning on Openshift using the attached template.
oc process -f https://raw.githubusercontent.com/osa-ora/simple_java_gradle/main/cicd/sonarqube-persistent-template.yaml | oc create -f -

Open SonarQube and create new project, give it a name, generate a token and use them as parameters in our next CI/CD steps

<img width="945" alt="Screen Shot 2021-01-03 at 16 07 57" src="https://user-images.githubusercontent.com/18471537/103480605-e630c880-4ddd-11eb-91de-c43f2882da84.png">

Make sure to select Gradle here.

## 4) Build Jenkins CI/CD using Jenkins File

Now create new pipeline for the project, where we checkout the code, run unit testing, run sonar qube analysis, build the application, get manual approval for deployment and finally deploy it on Openshift.
Here is the content of the file:

```
pipeline {
	options {
		// set a timeout of 20 minutes for this pipeline
		timeout(time: 20, unit: 'MINUTES')
	}
    agent {
    // Using the gradle builder agent
       label "gradle"
    }
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: '${git_url}'
        sh "ls -l"
      }
    }
    stage('Unit Testing') {
      steps {
        sh "gradle test"
      }
    }
    stage('Sonar Qube') {
      steps {
        sh "gradle sonarqube -Dsonar.login=${sonarqube_token} -Dsonar.host.url=${sonarqube_url} -Dsonar.projectKey=${sonarqube_proj}"
      }
    }
    stage('Build App'){
        steps{
            sh "gradle bootjar"
        }
    }
    stage('Deployment Approval') {
        steps {
            timeout(time: 5, unit: 'MINUTES') {
                input message: 'Proceed with Application Deployment?', ok: 'Approve Deployment'
            }
        }
    }
    stage('Deploy To Openshift') {
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
       label "gradle"
    }
```

The pipeline uses many parameters:
```
- String parameter: proj_name //this is Openshift project for the application
- String parameter: app_name //this is the application name
- String parameter: git_url //this is the git url of our project, default is https://github.com/osa-ora/simple_java_gradle
- String parameter: sonarqube_url: //this is the sonarqube url in case it is used for code scanning
- String parameter: sonarqube_token //this is the token 
- String parameter: sonarqube_proj // the project name in sonarqube
```

<img width="1270" alt="Screen Shot 2021-01-03 at 15 47 50" src="https://user-images.githubusercontent.com/18471537/103480534-92be7a80-4ddd-11eb-96b2-23007d19c242.png">

The project assume that you already build and deployed the application before, so we need to have a Jenkins freestyle project where we initally execute the following commands in Jenkins after checkout the code:
```
gradle bootjar
oc project ${proj_name}
oc new-build --image-stream=java:latest --binary=true --name=${app_name}
oc start-build ${app_name} --from-dir=build/libs/.
oc logs -f bc/${app_name}
oc new-app ${app_name} --as-deployment-config
oc expose svc ${app_name} --port=8080 --name=${app_name}
```
This will make sure our project initally deployed and ready for our CI/CD configurations, where proj_name and app_name is Openshift project and application name respectively.

You can enrich the project and add approvals to deploy to differnet environments as well, all you need is to have the privilages using "oc policy" as we did before and add deploy stage in the pipeline step.

