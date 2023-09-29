# Introduction
This document explains on how an application is deployed to AKS using the Jenkinsfile (declarative) pipeline.

## Jenkins Pipeline Stages

<img src="../docs/pipeline-stages.png">

## DAST Scan
The automated DAST will only cover a basic user interface based security test of the application. This test is sufficient for all internal application deployment to PROD, subjected to reivew & approval. All internet or customer facing applications must also performa a manual DAST to be ready for PROD deployment. 

However, this basic test and its integration with the CI/CD pipeline will help to fix the fundamental secrutiy issues identified in the application and reduce the work to do once manual test results are publisher. More details of the DAST ZAP setup can be found in the links below.

- Login to DPDHL Forge site - https://esharenew.dhl.com/sites/DPDHL_Forge/Tool/default.aspx
- Choose All Tools Option that navigates to Tools menu - https://esharenew.dhl.com/sites/DPDHL_Forge/Tool/Lists/Tool/Ordered%20by%20Category.aspxt
- Search for OWASP ZAP and click on the tool to know more. 

### Prerequisite
A request should be submitted to create a new DAST (Dynamic Application Security Testing) Jenkins job.

- Navigate to ITS1 Portal and choose projects - https://its1.dhl.com/group/guest/project-hub/projects
- Choose the project and click on "Request or Link tool". Project will be shown only for application team members.
- Choose OWASP ZAP and fill in the required fileds in the popup. 
  - Sample Application URL = https://dhl-app-fe-uat.dhl.com/
  - Sample Application Context Git Location: https://git.dhl.com/dhl-app-123/app-ui/tree/aks-migration/dast/
  - Sample Application Context File name: app-context-uat.xml
  - Sample Application Context Name: app-context-uat
- Refer image below for more details.

  <img src="../docs/dast-request.png">

### Jenkins Job Customization
The Jenkins job created via the ITS1 request will have application URL, context file location, context file name and context user values hard coded in the pipeline definition. This must be customized to remove the default values and accept values as input parameters which should be supplied at runtime by the invoking parent jenkins jobs, like the one in the next section. 

- Job without customizations 
  ```
  stage('DAST') {
    // Environment variables defined by requestor    
    env.CONTEXT_FILE= 'app-fe/dast/app-context-uat.xml'
    env.CONTEXT_NAME= 'dhl-app-fe'
    env.CONTEXT_USER_NAME= 'dhl-user'
    env.URL2SCAN= 'https://dhl-app-fe-uat.dhl.com/'
  ```
- Job after customization 
  ```
  stage('DAST') {
    // Environment variables defined by requestor    
    env.CONTEXT_FILE= params.CXT_FILE
    env.CONTEXT_NAME= 'dhl-app-fe'
    env.CONTEXT_USER_NAME= params.CXT_USER 
    env.URL2SCAN= params.CXT_URL
  ```

### Other Customizations
This DAST job executes the test and generates the report and does not fail the build in case of issues. Hence a scrip to identify critical issues if any should be added to the job execution stage before the POST stage actions. Based on the evaluated paramters the job should be marked a failure when critical issues are identified. 

```
nodeContainer([image(alias: 'zap', imageName: 'docker.artifactory.dhl.com/owasp/zap2docker-weekly')]) {
  container('zap') {
    ...
    ...
	  env.hasCriticalIssues = sh(script: 'cat report.json | grep -i riskdesc | grep -i high | wc -l', returnStdout:true)
  }
  archiveArtifacts artifacts: '*.html, *.xml, *.json'
    ... 
  if(null != hasCriticalIssues && hasCriticalIssues.length() >0 ){
		error("DAST ZAP Scan has identified #${hasCriticalIssues} critical vulnerabilities that needs action")
	}
    ...
}
```

### Jenkins Pipeline Integration 
The application's Jenkins pipeline should use the below code to invoke the external DAST Jenkins job. This stage and the Jenkins job will fail if there are any cricial issues idenfieid. 
The development team should review the failures and apply necessary code fix. Until the fix is deployment the pipeline may temporarily discard the errors and move forward with the deployment. 

```
stage('DAST & Scan via ZAP') {
  when {
    expression { 'dev' == params.appEnv || 'test' == params.appEnv || 'uat' == params.appEnv}
  }
  
  steps {
    build job: "../DAST_Scan/Express/DHL-APP/2022-10-05_1.0_EX-Public-Cloud-Migr_PJ20221437/DAST_Scan_PJ20221437",
      parameters: [[$class: 'StringParameterValue', name: 'ContextCreds', value: 'dhl-app-ssh-key'],
        [$class: 'StringParameterValue', name: 'CXT_FILE', value: "app-fe/dast/app-context-${appEnv}.xml"],
        [$class: 'StringParameterValue', name: 'CXT_USER', value: 'dhl-app-user'],
        [$class: 'StringParameterValue', name: 'CXT_URL', value: "https://dhl-app-fe-${appEnv}-cloud.dhl.com/"]
      ]
  }
 }
```
