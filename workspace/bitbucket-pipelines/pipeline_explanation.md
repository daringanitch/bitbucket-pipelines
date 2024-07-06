
# Pipeline Explanation

This pipeline file is used to define a CI/CD process that includes code quality analysis, building, and deploying a Maven project. Below is a detailed explanation of each section and step.

## Image
```yaml
image: maven:3.6.3-jdk-13
```
This specifies the Docker image to be used for the pipeline. In this case, it's the Maven 3.6.3 image with JDK 13.

## Definitions

### Caches
```yaml
caches:
  sonar: ~/.sonar/cache
```
Defines a cache for Sonar, which helps speed up the SonarCloud scan process by reusing previously downloaded dependencies.

### Services
```yaml
services:
  docker:
    memory: 3072
```
Defines a Docker service with 3072 MB of memory allocated. This service is required for running Docker commands in the pipeline.

### Steps
#### Code Quality Analysis with SonarCloud
```yaml
- step: &sonar-cloud
    name: Code Quality Analysis with SonarCloud
    caches:
      - sonar
    script:
      - pipe: sonarsource/sonarcloud-scan:2.0.0
        variables:
          SONAR_TOKEN: $SONAR_TOKEN
          EXTRA_ARGS: '-Dsonar.exclusions=**/*.java'
```
This step runs a SonarCloud scan for code quality analysis. It uses a cache for Sonar dependencies and excludes all `.java` files from the scan. The `SONAR_TOKEN` is used for authentication.

#### Build Jar File
```yaml
- step: &build-jar
    name: Build jar file
    script:
      - yum install python3-pip -y
      - pip3 install awscli --upgrade
      - export CODEARTIFACT_AUTH_TOKEN=$(aws codeartifact get-authorization-token --domain $DOMAIN --domain-owner $AWS_ACCOUNT_ID --query authorizationToken --output text)
      - echo $CODEARTIFACT_AUTH_TOKEN
      - sed -i "s/EXECUTION_SERVICE_VERSION/$PACKAGE_VERSION/" pom.xml
      - mvn clean package -s settings.xml -e -DskipTests # Compiles and packages the project
      - mvn deploy -s settings.xml -DskipTests
```
This step installs necessary dependencies, retrieves an authorization token from AWS CodeArtifact, updates the `pom.xml` file with the package version, and then builds and deploys the Maven project.

#### Deploy to EC2
```yaml
- step: &deploy-to-ec2
    name: Deploy to EC2
    script:
      - yum install python3-pip jq -y
      - pip3 install awscli --upgrade
      - ASSET_NAME=$(aws codeartifact list-package-version-assets --region $REPO_REGION --domain $DOMAIN --domain-owner $AWS_ACCOUNT_ID --repository execution-service --format maven --namespace com --package executionservice --package-version $PACKAGE_VERSION | jq .assets[0].name | tr -d '"')
      - aws ssm send-command --targets "Key=instanceids,Values=$INSTANCE_IDS" --document-name "AWS-RunShellScript" --parameters "commands=aws codeartifact get-package-version-asset --region $REPO_REGION --domain $DOMAIN --domain-owner $AWS_ACCOUNT_ID --repository execution-service --format maven --namespace com --package executionservice --package-version $PACKAGE_VERSION --asset $ASSET_NAME $SERVICE_DIR/execution-service/execution-service.jar,sh $SERVICE_DIR/execution-service/executionrestart.sh"
```
This step installs additional dependencies, retrieves the asset name from AWS CodeArtifact, and uses AWS Systems Manager (SSM) to send a command to EC2 instances to download and deploy the package.

## Stages

### Sonar Cloud Stage
```yaml
- stage: &sonar-cloud-stage
    name: Sonar Cloud
    steps:
      - step: *sonar-cloud
```
This stage runs the SonarCloud scan for code quality analysis.

### Deploy to Development Stage
```yaml
- stage: &deploy-to-dev-stage
    name: Deploy to Development
    deployment: Development
    steps:
      - step: *build-jar
      - step: *deploy-to-ec2
```
This stage includes steps to build the jar file and deploy it to a development environment.

### Deploy to Staging Stage
```yaml
- stage: &deploy-to-uat-stage
    name: Deploy to Staging
    deployment: Staging
    steps:
      - step: *build-jar
      - step: *deploy-to-ec2
```
This stage includes steps to build the jar file and deploy it to a staging environment.

### Deploy to US Production Stage
```yaml
- stage: &deploy-to-us-prod-stage
   name: Deploy to US Prod
   deployment: US Prod
   steps:
     - step: *deploy-to-ec2
```
This stage includes a step to deploy the package to a US production environment.

### Deploy to UK Production Stage
```yaml
- stage: &deploy-to-uk-prod-stage
   name: Deploy to UK Prod
   deployment: UK Prod
   steps:
     - step: *deploy-to-ec2
```
This stage includes a step to deploy the package to a UK production environment.

### Deploy to IND Production Stage
```yaml
- stage: &deploy-to-ind-prod-stage
   name: Deploy to IND Prod
   deployment: IND Prod
   steps:
     - step: *deploy-to-ec2
```
This stage includes a step to deploy the package to an IND production environment.

## Pipelines

### Default Pipeline
```yaml
default:
  - stage: *sonar-cloud-stage
```
The default pipeline runs the SonarCloud scan.

### Custom Pipelines
#### Development Pipeline
```yaml
custom:
  dev:
    - variables:
        - name: PACKAGE_VERSION
          type: string
    - stage: *deploy-to-dev-stage
```
The development pipeline allows for specifying a `PACKAGE_VERSION` and includes steps to build and deploy the package to a development environment.

#### Staging Pipeline
```yaml
staging:
  - variables:
      - name: PACKAGE_VERSION
        type: string
  - stage: *deploy-to-uat-stage
```
The staging pipeline allows for specifying a `PACKAGE_VERSION` and includes steps to build and deploy the package to a staging environment.

#### US Production Pipeline
```yaml
us-prod:
  - variables:
      - name: PACKAGE_VERSION
        type: string
  - stage: *deploy-to-us-prod-stage
```
The US production pipeline allows for specifying a `PACKAGE_VERSION` and includes a step to deploy the package to a US production environment.

#### UK Production Pipeline
```yaml
uk-prod:
  - variables:
      - name: PACKAGE_VERSION
        type: string
  - stage: *deploy-to-uk-prod-stage
```
The UK production pipeline allows for specifying a `PACKAGE_VERSION` and includes a step to deploy the package to a UK production environment.

#### IND Production Pipeline
```yaml
ind-prod:
  - variables:
      - name: PACKAGE_VERSION
        type: string
  - stage: *deploy-to-ind-prod-stage
```
The IND production pipeline allows for specifying a `PACKAGE_VERSION` and includes a step to deploy the package to an IND production environment.
