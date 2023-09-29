# Introduction
This repository contains AKS - Helm deployment chart templates for widely used application technology stacks.

Table of Contents

<!-- TOC -->

- [AKS Overview and Prerequisites](#aks-overview-and-prerequisites)
  - [AKS Migration Workflow](#aks-migration-workflow)
  - [Azure Security Group](#azure-security-group)
  - [Azure Service Principal](#azure-service-principal)
  - [AKS Cluster](#aks-cluster)
  - [AKS Namespaces](#aks-namespaces)  
  - [AKS Authentication](#aks-authentication)
  - [Azure Container Registry](#azure-container-registry)
- [Application Prerequisites](#application-prerequisites)
  - [GitHub Credential Setup](#github-credential-setup)
  - [Artifactory Credential Setup](#artifactory-credential-setup)
  - [DNS Setup](#dns-setup)
  - [SSL Certificates](#ssl-certificates)
  - [External Link Request](#external-link-request)
- [ASK Entities](#aks-entities)
  - [Configmap](#configmap)
  - [TLS Secret](#tls-secret)
  - [Storage Secret](#storage-secret)
  - [Image Pull Secret](#image-pull-secret)
  - [Deployments and PODs](#deployments-and-pods)
    - [Single Pipeline](#single-pipeline)
    - [Split Pipeline](#split-pipeline)
- [Splunk Integration](#splunk-integration)
  - [Procedure to setup custom index](#procedure-to-setup-custom-index)
  - [Splunk Annotations](#splunk-annotations)
  - [Splunk URLs by Environment](#splunk-urls-by-environment)
- [Oracle Cloud Dependency](#oracle-cloud-dependency)
- [Accessing AKS Cluster](#accessing-aks-cluster)
  - [From Developer Laptops](#from-developer-laptops)
  - [Useful kubectl Commands](#useful-kubectl-commands)
- [Docker Images](#accessing-docker-images)
  - [Accessing DHL Artifactory](#dhl-artifactory)
  - [Accessing Azure Container Registry](#azure-container-registry)
  - [Docker Desktop for Developer](#docker-desktop-for-developers)
<!-- /TOC -->

# AKS Overview and Prerequisites

<img src="docs/aks-cluster-overview.jpg">

## AKS Migration Workflow

Application teams will follow the steps below in sequence to execute the AKS migration. 

| S.No | Request Type | Description | Team Responsible |
| --- | --- | --- | --- |
| 1 | Service Now | Create Azure AD Security Group and get Group owners assigned | Global MDS Team |
| 2 | Service Now | request to create Azure Service Principal with Push/Pull privilege to application Azure Container Registry (ACR) | SPCS PCCP Team |
| 3 | Service Now | request to create AKS Cluster with namespaces (dev, test, uat etc) associated with the compute nodepool along with Contributor privileges to the Azure AD Security group created in #1 | SPCS PCCP Team |
| 4 |  |Azure AD Group owner to add/remove Users (App team members) and Azure Service Principal who can inherit access/privilge to AKS namespaces | Application Team | 
| 5 |  | Verfiy access to AKS namespace via Azure Cloud Shell or local shell in Developer laptops | Application Team | 
| 6 | CMC Portal Request | Create SSL SAN certificates after concluding on the domain names | Application Team | 
| 7 | Service Now | Create DNS CNAME entries for application domains | NOC Team |
| 8 | Service Now | Create custom Splunk Index for efficient log and metrics search for each namespace | SPCS Monitoring team |
| 9 | Service Now | Configure splunk index, token and collector URL in AKS Namespace & configure the index to helm templates | SPCS PCCP team |
| 10 | Service Now | Create/migration Database from on-prem to cloud | SPCS DB PaaS team |
| 11 | Service Now / ELR Process | Implement ELRs to open up connectivity from Azure to On-prem components | SAO/ELR Team |
| 12 | Service Now / ELR Process | Implement B2B Proxy ELRs to open up connectivity to services external to DHL | SAO/ELR team |
| 13 |  | Deployment of application pre-requisites - Configmaps, AKS Secrets for files, config & OCI Wallet, TLS Secret | Application Team | 
| 14 |  | Deployment of application contianers - Deployment of PODs, Cronjobs, Service and Ingress | Application Team |


## Azure Security Group

Application teams will be granted Admin & ReadOnly access to their namespaces in the AKS Cluster via respective Azure Active Directory Secruity groups. Hence each application team should create appropriate security groups to get this access. 

A service now request should be submitted to MDS team to create the Azure Active Directory Security Groups along with a note to assign the appropriate team members as owners of the group. This group is snychorized to Azure and will be used by SPCS team to grant contributor privileges to the AKS Cluster/Namespace. 

Azure Active Directory Security Group owners (Product Owners/Mangers) can add/remove application team members to the appropriate group to grant/revoke relevant privileges to the application team members via Azure Portal. 

The Azure Service Principal should be added as a member of the admin security group with contributor privileges and will be used by Jenkins jobs/scripts for deployment purposes.

## Azure Service Principal 

Application team should request for Azure Service Principal for the CI/CD job execution. This service principal contains the below attributes. 
1. Service Principal Name: SP-EXP-BI-APP-DEV-001
2. Application ID: 2445d27b-xxxx-xxxx-xxx-xxxxxxxx07346 - Referred as Client ID in the CI/CD scripts
3. Object ID: 3b74c390-xxxx-xxxx-xxx-xxxxxxxxc8fd9 - 
4. Directory ID: cd99fef8-xxxx-xxxx-xxx-xxxxxxxx1d65e   - Referred as Tenant ID in the CI/CD scripts
5. Secret: 2gZ8Q~-xxxx-xxxx-xxx-xxxxxxxxAWubQb (Validity 6 months)  - Referred as Client Secret in the CI/CD scripts

It is strongly recommended to add the service principal to Jenkins Credentials (Type: Azure Service Principal) and verify the same using the option in the credentials page.

Request to create an Azure Service Principal can be submitted here -https://dpdhl.sharepoint.com/sites/DEnablement/DWS/IAM/App/SSO/AAD%20Enterprise%20Application%20and%20Service%20Principal%20OnBoarding.aspx. 

## AKS Cluster

SPCS PCCP team (Secure Public Cloud Services - Public Cloud Container Platform) manages all the Azure resoruces. Domain specific clusters are already created by this group in the nonprod (ITS_Cloud_Container_Platform-9099) and prod (ITS_Cloud_Container_Platform-Prod-9098) subscriptions. 

Application teams will have to submit a SNOW (ServiceNOW) request to this team to create their environment specific namespaces along with their AD Security group(s) and Azure Service Principal to get necessary privileges. 

- SNOW request to create AKS Namespaces
- SNOW request to grant privileges to AD Security Group(s)
- SNOW to grant ACR privilges to Azure Service Principal 

## AKS Namespaces

All non prod environments - Dev, Test, UAT & SIT  will be using the same AKS Cluster in NonPROD and and the PROD environment will be hosted in a separate cluster. These environements are logically seperated by Namespaces that are created in the format - exp-<domain>-<app/product>-<environment> as listed below.
	
- exp-bi-app-dev - Namespace that contains deployments of the app in development environment
- exp-bi-app-test - Namespace that contains deployments of the app in test environment
- exp-bi-app-sit - Namespace that contains deployments of the app in simple integration environment
- exp-bi-app-uat - Namespace that contains deployments of the app in UAT environment

## AKS Authentication

Azure Service Principal is stored as "Service Principal" credential in the Jenkins Credential store and used to authenticate to the Azure Container Registry.
```
withCredentials([azureServicePrincipal("JENKINS_SERVICE_PRINCIPAL_CREDENTIAL_ID")]) {
  docker login itsacrexpbi.azurecr.io -u ${AZURE_CLIENT_ID} -p ${AZURE_CLIENT_SECRET}
}
```
The same Azure Service Principal credential's Client ID, Client Secret and Tenant ID are used to peform AZ login followed by kubelogin to authenticate to the AKS Cluster and access the namespace. 

If the same Azure SP is granted access to more than one Azure Subscription (like non prod and prod cluster), please ensure to set the right subscription before executing kubectl  or helm commands (az account set -s <subscriptionId>) part of deployment stage. 

Verify Service Principal using Jenkins script or a pipeline stage:

```
withCredentials([azureServicePrincipal("JENKINS_SERVICE_PRINCIPAL_CREDENTIAL_ID")]) {
  
  az login --service-principal -t "${AZURE_TENANT_ID}" -u "${AZURE_CLIENT_ID}" -p "${AZURE_CLIENT_SECRET}"
  az account set -s "${AZURE_SUBSCRIPTION_ID}"
  az account show
  
  az aks get-credentials --resource-group ${AKS_RG} --name ${AKS_CLUSTER}
						
  kubelogin convert-kubeconfig -l spn
	
  export AAD_SERVICE_PRINCIPAL_CLIENT_ID="${AZURE_CLIENT_ID}"
  export AAD_SERVICE_PRINCIPAL_CLIENT_SECRET="${AZURE_CLIENT_SECRET}"
  
  kubectl -n ${AKS_NAMESPACE} get pods
  helm upgrade --install ${AKS_APP_NAME} ./deployment -f ./${appEnv}-values.yaml --namespace=${AKS_NAMESPACE} --history-max 3
}
```

## Azure Container Registry

Application docker images should be stored to the domain specific Azure Container registries that already exists. Refer to the table below for the complete list of Azure Container registries for DHL Express. 

| Domain | Container registry | Resource Group |
|---|---|---|
| BI - Business Intelligence | itsacrexpbi.azurecr.io | RG-AKS-EXP-BI-DEV-WEU-001 |
| CFIT - Customer Facing IT | itsacrexpcfit.azurecr.io | RG-AKS-EXP-CFIT-DEV-WEU-001 |
| FIN - Finance | itsacrexpfin.azurecr.io | RG-AKS-EXP-FIN-DEV-WEU-002 | 
| HR - Human Resources | itsacrexphr.azurecr.io | RG-AKS-EXP-HR-DEV-WEU-001 |
| NDAT - Network Data | itsacrexpndat.azurecr.io | RG-AKS-EXP-NDAT-DEV-WEU-001 |
| NOPS - Network Operation | itsacrexpnops.azurecr.io | RG-AKS-EXP-NOPS-DEV-WEU-001 |
| SI - System Integration | itsacrexpsi.azurecr.io | RG-AKS-EXP-SI-DEV-WEU-001 |
| SM - Sales & Marketing | itsacrexpsm.azurecr.io | RG-AKS-EXP-SM-DEV-WEU-001 |

A request should be made to SPCS team to grant ACR (Azure Container Registry) Pull & Push privileges to the Azure Service Principal. This Service Principal is used by the Jenkins jobs to store application docker images to ACR and by AKS to pull the docker images to run the application service during deployment.

# Application Prerequisites

## GitHub Credentials Setup

A GitHub SSH Key has been generated using the linux command given below. This command produces a public and private key pair that can be used by Jenkins Job to authenticate with GitHub and checkout the source code.

```ssh-keygen -t ed25519 -C "<serviceaccount>@dhl.com"```

The generated public key is copied and configured as the trusted keys of the application's service account profile in GitHub. The private key is added to Jenkins credentials and used by the jobs for SCM Checkout. 

## DHL Artifactory Credentials Setup

Similarly, after login to Artifactory with the service account, an Identity token has been generated, which is valid for 1 year and the same has been configured in place of password in the settings.xml used by maven build for dependency resolution. Another alternative is to use the encrypted password as a runtime secret to the Jenkins jobs.

## DNS Request

Application teams prefer to setup a separate domain name for AKS environment without disturbing their present OpenShift environment domain name/URL. A service now request should be submitted to the network team (NOC Team) to create the DNS entries mapped to the CNAME of the AKS Cluster. 

All the non-prod AKS namespaces will have same ingress mapping whereas PROD AKS will have a different done. Based on the domain name of the application individual CNAME records for each env or a wilcard CNAME record can be created as given below.

- app-dev.dhl.com    azure-aks-exp-bi-dev-weu-001-01.dhl.com
- app-test.dhl.com    azure-aks-exp-bi-test-weu-001-01.dhl.com
- app-e2e.dhl.com    azure-aks-exp-bi-e2e-weu-001-01.dhl.com

For an application with environment as subdomains like dev.app.dhl.com, test.app.dhl.com a single wildcard CNAME entry would suffice. 

- *.app.dhl.com    azure-aks-exp-bi-dev-weu-001-01.dhl.com

## SSL Certificate

Compared to classic single SSL certificate, a SSL SAN certificate can accomodate multiple domain names via Subject Alternate Names within the same certificate and offer exactly same SSL security (https) for applications. Hence a SSL SAN certificate with one primary domain name as CN/Subject and all other Non-PROD domain name as SANs is recommended for applications. SAN certificate save a huge maintenance effort each year during the SSL renewal. SSL Certificate can be generated using the below steps. 

1. Prepare the <a href="docs/csrconfig.txt">CSR configuration file</a> as given below

```
[alt_names]
DNS.1 = app-be-preprod-cloud.dhl.com
DNS.2 = app-fe-test-cloud.dhl.com
DNS.3 = app-fe-uat-cloud.dhl.com

[req_distinguished_name]
commonName = Common Name (domain to be signed, e. g. example.dhl.com)
commonName_default = app-fe-dev-cloud.dhl.com
```

3. Execute the below command to generate private key and CSR: (Passphrase shall be given or left empty )

```openssl req -new -newkey rsa:2048 -keyout app-np.key -config csrconfig.txt -out app-np.csr```

3. Create a SNOW ticket to request SSL by uploading the Key and CSR file

```https://cmc.dhl.com/tlsCertificates/request```

4. After receiving the SSL Cert, decrypt the Private key using command below. This decrypted key is used to create the TLS Secret in AKS. 

```openssl rsa -in app-np.key  -out app-np-decrypt.key```
	
Refer to the TLS Secret section to know more about trasnforming the SSL Cert to an AKS TLS Secret and utilize the same in AKS Ingress. 

## External Link Request (ELR)

By default outbound access from AKS to onPrem DHL network is disabled in the firewall settings. In order to access any components in the DHL networking ELR request must be submitted to the ELR team with AKS nodepool's source IP CIDR Range and destination IP/hostname and Port. 

Some of the DHL components work behind a load balancer and ELRs implemented with this load balancer's IP address will not work or will work inconsistently. Hence, it is recommended to get the list of all backend IP addresses behind the load balancer and share them with ELR team for a complete ELR implementation.

Application that needs to connect to services external to DHL have couple of options
- If the application has capability to use http proxy, application can request for a B2B proxy ELRs to connect to the external services
- Otherwise, application team has be raise request for Firewall exemption with CISO & Service Owner Approval. 

	
On successful implementation of the ELRs, application will be able to connect to the requested compoenet. An networking tools pod can be created in each namespace to run the networking commands like ping, telnet, nsloopup, tracert to verify the ELR implementation to the destinations. Networking POD deployment yaml (networking-tools.yaml) is given below for reference.
	
```kubectl apply -f networking-tools.yaml```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: networking-tools
  namespace: exp-bi-eot-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: networking-tools
  template:
    metadata:
      labels:
        app: networking-tools
    spec:
      automountServiceAccountToken: false
      containers:
      - name: networking-tools
        image: itsacrexpbi.azurecr.io/core/networking-tools:v1.0.1 ## USER your DOMAIN's ACR
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        securityContext:
          privileged: false
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 5000
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
```

# AKS Entities

## Configmap

Config maps are used to store configuration 
- key/value pairs
- files (yaml, properties)
that are be used by the application being deployed. 
```
data:
  DATABASE_HOST: oracle.dhl.com:1521
  application.yml: |-
    app:
      location: PRG
      feign:
        timeout: 360000
```
These config map key/value pairs are are loaded as environment variables (envFrom) and the config files from the config maps are mounted a file accessible as a volume in to the application contianer during deployment.
```
envFromConfigMap:
  - configMap: app-db-config
    key: DATABASE_HOST
    
configVolumes:
  - name: db-config
    configMapName: app-db-config
    key: application.yml
    path: application.yml
    mountPath: /config/application.yml
```
A separate Jenkins job shall be developed to create/update config map and its values without impacting the application deployment. 
<br/><br/>
Refer app-[app-config-secrets/config-map](/app-config-secrets/config-map) for the sample templates. 

## TLS Secret

The SSL Private Key (decrypted) and Certificate are configured as Jenkins credentials of type "secret-file". This credential is pushed as TLS secret to each AKS namespace and refered by the ingress modules for SSL handshake and processing. 
<br/><br/>
Refer to [app-config-secrets/tls-secret](/app-config-secrets/tls-secret) for sample template. 

Below command shall be used verify successful push of the TLS secret by fetching it expiry date.
```kubectl get secret app-nonprod -o "jsonpath={.data['tls\.crt']}" | base64 --decode | openssl x509 -enddate -noout```

## Image Pull Secret

The Client ID and Secret value, in an Username: password format, of the Azure Service Principal (Azure SP) is pushed as a Image Pull Secret to each AKS namespace. This secret is used by the pods to fetch docker image from Azure ACR during application startup. 
<br/><br/>
Refer [app-config-secrets/img-secret](/app-config-secrets/img-secret) for sample template.

## Storage Secret

Storage secret is primarily used to access an Azure File share and mount it to a container. An access generated for the storage account via Azure Portal is configured as the storage secret in AKS. Application containers will use this secret to access the Azure file share and mount it as a peristent local storage. 
<br/><br/>
Refer [app-config-secrets/storage-secret](/app-config-secrets/storage-secret) for sample template.

## Deployments and Pods

Each application module has been defined through a helm library based deployment template and deployed to AKS Namespace. These deployments scale up immediately after deployment in to runtime containers called Pods. HPA - Horizontal Pod Autoscaler is assocaited with a deployment to enable autoscaling capabilites. 

### Single Pipeline
Build and deployment stages/steps are scripted together as a single deployment pipeline for simple application. This will deploy the configmap, secret, ingress, service and deployment entitles together in the same deployment.
<br/><br/>
Refer to [java-maven-helm](/java-maven-helm) template for more details. 

### Split Pipeline
Continuous Integration (CI) steps such as static code analysis, code quality & vulnerability check followed by compliation, build packaging and docker image creation are scripted together as Jenkins pipeline for complex applications having multiple modules to be deployed. Similary, Continuous Deployment (CD) is implemented via the helm library based Jenkins pipeline script.
All CI jobs of the application will invoke the same CD job with appropriate input parameters for AKS deployment.
<br/><br/>
Refer to [ci-cd-split-pipelines](/ci-cd-split-pipelines) template for more details. 


# Splunk Integration

Splunk is the preferred enterprise tool for AKS monitoring, Log aggregation and Alerting (Emails). 
	
A generic index is available by default for all application namespaces in AKS to which logs and metrics are pushed. The preserved information can be accessed using the URLs below. Search through this generic default index may lead to increased processing time while querying logs or metrics for the application. 

Hence SPCS Monitoring team recommends application teams to create their own index for each namespace to segregate data and make search easy. There is NO ADDITIONAL COST involved in creating custom indexes apart from the regular data ingestion changes. 

## Procedure to setup custom index

- Submit a request to create custom splunk indexes for each namespace or consolidated index for prod/non-prod namepsaces using the DHL Form - https://gsd.dhl.com/forms/5374
- SPCS Monitoring team will create the indexes and share index name, token and splunk API URL.
- Configure the index name in the application dpeloyment as splunk annotation, refer splunk annotations sample below
- Create a SNOW request to SPCS PCCP team with token and Splunk API URL to configured in the Splunk Collector Agent so that they push the metrics and logs to the relevant splunk instance. 
- Redeploy application after which logs and metrics should be available in splunk console.
	
## Splunk Annotations

```
## Annotations for metrics and logs
annotations:
  collectord.io/index: kubernetes_aks_exp_domain_dev_weu_001
  collectord.io/logs-output: splunk
```

```
## Annotations for logs from physical file
annotations:
  collectord.io/index: kubernetes_aks_exp_domain_dev_weu_001
  collectord.io/logs-output: splunk
  collectord.io/volume.1-logs-name: vol-app-log
  collectord.io/volume.1-logs-glob: '*.log'
  collectord.io/volume.1-logs-index: kubernetes_aks_exp_domain_dev_weu_001
  collectord.io/volume.1-logs-output: splunk
  collectord.io/volume.1-logs-type: app-log
```

## Splunk URLs by Environment

| Environemnt | Splunk URL |
| --- | --- |
| EU - WEU | https://dhl-eu.splunkcloud.com/en-US/app/monitoringkubernetes/search |
| AP - SEA | https://dhl-ap.splunkcloud.com/en-US/app/monitoringkubernetes/search |
	
# Oracle Cloud Dependency

Applications using Oracle database have couple of options to migrate to AKS. They can migrated to PostGRES database or Oracle Database on cloud - OCI. Request should be submitted to SPCS Cloud PaaS DBA team for further assistance. 
	
OCI database can be accessed only with the specical database connection string and the OCI wallet files which will be provided by the SPCS team. The connection string will be of format given below. 

```
jdbc:oracle:thin:@exp<domain><app><env>_high?TNS_ADMIN=/<wallet_folder>
```
The OCI wallet files should be pushed as an AKS Secret and that should be mounted to the application contianers as volume. This volume location should be suffixed in place of the wallet folder to the database connection URL. 

Refer to [oci-wallet](/app-config-secrets/oci-wallet) for a sample pipeline to create OCI wallet in AKS with the wallet zip file from Jenkins credentials store. 
	
In addition, springboot based applications would need 3 OCI dependency jar files to be included final package for the application to establish connection to the database. 

```
<!-- OCI Secutiy JAR Dependencies -->
<dependency>
      <groupId>com.oracle.database.security</groupId>
      <artifactId>osdt_cert</artifactId>
      <version>21.1.0.0</version>
</dependency>
<dependency>
      <groupId>com.oracle.database.security</groupId>
      <artifactId>osdt_core</artifactId>
      <version>21.1.0.0</version>
</dependency>
<dependency>
      <groupId>com.oracle.database.security</groupId>
      <artifactId>oraclepki</artifactId>
      <version>21.1.0.0</version>
</dependency> 
```

# Accessing AKS Cluster]
	
AKS cluster can be accessed using couple of options 
- Azure Cloud Shell 
- Developer Laptops/VMs. 

## Azure Cloud Shell
	
Login to https://portal.azure.com and launch the Cloud Shell using icon available in nav bar next to the search field. This would beed a Azure Storage -> file share to be configured to preserve commands being executed and to store any temporary files. Choose "ITS_Cloud_Container_Platform-9099" as the subscription and then choose the application's AKS Resource Group to locate the default "cloudshell" storage account. Developers will have privilege to create their own fileshare under respective storage account.

## From Developer Laptops
	
Alternatively, AZ Cli commands can be installed to developer laptops with help from the service desk. After this the regular command prompt or git bash console can be used to execute AZ Cli commands after setting couple proxies listed below. 

```
set HTTP_PROXY=http://localhost:9000
set HTTPS_PROXY=http://localhost:9000
```
These proxies would bypass the Zscaler and allow developers to perform AZ login followed exeuction of az cli commands. 

## Connecting to AKS Cluster

After performing AZ login, the below commands has to be executed to connect to the AKS cluster 

```
az account set --subscription <subscriptionID>  ##Set the Azure Subscription
az aks get-credentials --resource-group RG-AKS-**** --name AKS-EXP-***   ##Get the AKS Cluster Credentials
```

Above step will download the AKS credentials to the file "<userhome>/.kube/config". As Azure disabled auth plugin to execute kubectl commands, the bleow command must be executed to conver the ./kube/config file to use kubelogin based authentication. 

```
kubelogin convert-kubeconfig
```

After successful execution of above steps kubectl commands can be executed. 
	
## Useful AZ and kubectl Commands

Get Deployments list 
```
kubectl -n <namespace> get deployments
```
Get Pods list
```
kubectl -n <namespace> get pods
```
Get ConfigMaps list
```
kubectl -n <namespace> get configmap/cm
```
Get Secrets list
```
kubectl -n <namespace> get secrets
```
Get Services list
```
kubectl -n <namespace> get service/svc
```
Get Ingress list
```
kubectl -n <namespace> get ingress/ing
```
Get Ingress Pods list
```
kubectl -n dev-ext get pods
```
Get Pod logs
```
kubectl -n <namespace> logs <podname>
```
Get Ingress Pod logs
```
kubectl -n dev-ext logs <IngressPodname>
```

Scale up deployment pods 
```
kubectl -n <namespace> scale deployment <deploymentName> --replicas=1
```
Scale down deployment pods
```
kubectl -n <namespace> scale deployment <deploymentName> --replicas=0
```
	
Copy docker image from one ACR to another 
```
az acr import --name itsacrexpndat.azurecr.io --source itsacrexpnops.azurecr.io/core/networking-tools:v1.0.1 --image core/networking-tools:v1.0.1
```

Create a test cron job to exeucte it on demand without needing to wait for the schedule 
```
kubectl -n <namespace> create job <testJobName> --from=cronjob/<deployedCronJobName>
```

# Accessing Docker Images

Docker images used by CI tools like Jenkins are stored in the DHL Artifactory - https://docker.artifactory.dhl.com. The application specific docker images are stored in the Azure container registry to ease of download by AKS clusters. 

## Accessing DHL Artifactory

Docker base images like JDK images, Maven images, NPM images and more are availalbe in DHL artifactory. Example:  "docker.artifactory.cbj.dhl.com/adoptopenjdk/openjdk11:latest". Any image that is not available in the DHL artifactory but available in the public docker hub can be accessing using the DHL's proxy artifactory URL - https://docker-registry.artifactory.cbj.dhl.com/<imageSuffix>. Example: https://docker-registry.artifactory.cbj.dhl.com/tomcat:9.0-jdk17-corretto. 
	
DHL Artifactory can be authenticated using the regular LDAP credentials. Sample command is given below.

```
docker login docker.artifactory.dhl.com -u <ldapID> -p <ldapPassword>
```

## Accessing Azure Container Registry

Azure container registry will be accessible after AZ login wither with regular user credentials or Azure Service Principal. 

With Azure Service Principal: 
```
docker login ${AZURE_ACR} -u ${AZURE_CLIENT_ID} -p ${AZURE_CLIENT_SECRET}
```

Since Docker Dekstop v19 use following to login to Azure ACR (please note az cli must be installed):
```
docker login azure
```
and following for Service Principal:
```
docker login azure --client-id ${AZURE_CLIENT_ID} --client-secret ${AZURE_CLIENT_SECRET} --tenant-id ${AZURE_TENANT_ID}
```

For more information, refer https://docs.docker.com/cloud/aci-integration/

## Docker Desktop for Developer

Docker desktop tool can be installed on developer laptopn with support/approval from the service desk. This tool can help to download docker images from DHL Artifactory and Azure Container Registries to test or to perform Twistloc scan of the docker imagess.
	
Docker desktop tool would need the http_proxy and https_proxy to be set to https://localhost:9000 to bypass the Zscaler restrictions. 
	
The dockerfile below is an example where a NodeJS base is taken to install JDK8 and push it to DHL artifactory as a new application specific base docker image. 
```	
FROM docker-registry.artifactory.dhl.com/node:10.16.3
RUN apt-get update
RUN apt-get install -y openjdk-8-jdk
RUN npm -v && java -version
RUN npm install
```
