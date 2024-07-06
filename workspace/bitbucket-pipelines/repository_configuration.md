# Repository Configuration for CI/CD

To accommodate this pipeline file, the repository needs to be configured to support continuous integration and continuous deployment (CI/CD) with the specified tools and services. Here are the detailed steps to configure the repository:

## 1. Repository Structure

Ensure your repository has a proper structure, with all necessary files and directories. A typical Java/Maven project structure is as follows:

```
repository-root/
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
│       ├── java/
│       └── resources/
├── pom.xml
├── settings.xml
├── .pipeline/
│   └── pipeline-file.yml
└── README.md
```

## 2. Required Files

- `pom.xml`: The Maven project descriptor.
- `settings.xml`: Maven settings file, which might include repository configurations.
- Pipeline file: This should be saved in a directory like `.pipeline/` or at the root of the repository.

## 3. SonarCloud Configuration

1. **SonarCloud Account**: Create an account on SonarCloud and set up a project.
2. **SonarCloud Token**: Generate a SonarCloud token and store it securely.
3. **Configuration in `pom.xml`**: Add the Sonar plugin to your `pom.xml` if not already present:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.sonarsource.scanner.maven</groupId>
      <artifactId>sonar-maven-plugin</artifactId>
      <version>3.7.0.1746</version>
    </plugin>
  </plugins>
</build>
```

4. **Environment Variable**: Add `SONAR_TOKEN` to your repository's environment variables in the CI/CD settings.

## 4. AWS CodeArtifact Configuration

1. **AWS CLI**: Ensure the AWS CLI is configured on the machine running the pipeline.
2. **Environment Variables**: Set up the following environment variables:
   - `AWS_ACCOUNT_ID`
   - `DOMAIN`
   - `REPO_REGION`
   - `INSTANCE_IDS`
   - `SERVICE_DIR`

## 5. Deployment Configuration

1. **AWS Systems Manager (SSM)**: Ensure that the EC2 instances have the SSM agent installed and properly configured.
2. **IAM Roles**: Create and assign IAM roles with the necessary permissions for:
   - Accessing CodeArtifact
   - Executing SSM commands on EC2 instances
3. **Environment Variables**: Ensure the necessary environment variables are set for your deployments.

## 6. CI/CD Tool Configuration

### Bitbucket Pipelines (example)

If you are using Bitbucket Pipelines, place the pipeline file at the root of the repository as `bitbucket-pipelines.yml`. Here’s how you can set it up:

1. **Repository Settings**: Navigate to your repository settings in Bitbucket.
2. **Pipelines Settings**: Enable pipelines if not already enabled.
3. **Environment Variables**: Add the necessary environment variables under the repository settings:
   - `AWS_ACCOUNT_ID`
   - `DOMAIN`
   - `REPO_REGION`
   - `INSTANCE_IDS`
   - `SERVICE_DIR`
   - `SONAR_TOKEN`

## 7. Example `bitbucket-pipelines.yml`

Ensure your `bitbucket-pipelines.yml` reflects the structure provided earlier:

```yaml
image: maven:3.6.3-jdk-13

definitions:
  caches:
    sonar: ~/.sonar/cache
  services:
    docker:
      memory: 3072
  steps:
    - step: &sonar-cloud
        name: Code Quality Analysis with SonarCloud
        caches:
          - sonar
        script:
          - pipe: sonarsource/sonarcloud-scan:2.0.0
            variables:
              SONAR_TOKEN: $SONAR_TOKEN
              EXTRA_ARGS: '-Dsonar.exclusions=**/*.java'
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
    - step: &deploy-to-ec2
        name: Deploy to EC2
        script:
          - yum install python3-pip jq -y
          - pip3 install awscli --upgrade
          - ASSET_NAME=$(aws codeartifact list-package-version-assets --region $REPO_REGION --domain $DOMAIN --domain-owner $AWS_ACCOUNT_ID --repository execution-service --format maven --namespace com --package executionservice --package-version $PACKAGE_VERSION | jq .assets[0].name | tr -d '"')
          - aws ssm send-command --targets "Key=instanceids,Values=$INSTANCE_IDS" --document-name "AWS-RunShellScript" --parameters "commands=aws codeartifact get-package-version-asset --region $REPO_REGION --domain $DOMAIN --domain-owner $AWS_ACCOUNT_ID --repository execution-service --format maven --namespace com --package executionservice --package-version $PACKAGE_VERSION --asset $ASSET_NAME $SERVICE_DIR/execution-service/execution-service.jar,sh $SERVICE_DIR/execution-service/executionrestart.sh"

  stages:
    - stage: &sonar-cloud-stage
        name: Sonar Cloud
        steps:
          - step: *sonar-cloud
    - stage: &deploy-to-dev-stage
        name: Deploy to Development
        deployment: Development
        steps:
          - step: *build-jar
          - step: *deploy-to-ec2
    - stage: &deploy-to-uat-stage
        name: Deploy to Staging
        deployment: Staging
        steps:
          - step: *build-jar
          - step: *deploy-to-ec2
    - stage: &deploy-to-us-prod-stage
       name: Deploy to US Prod
       deployment: US Prod
       steps:
         - step: *deploy-to-ec2
    - stage: &deploy-to-uk-prod-stage
       name: Deploy to UK Prod
       deployment: UK Prod
       steps:
         - step: *deploy-to-ec2
    - stage: &deploy-to-ind-prod-stage
       name: Deploy to IND Prod
       deployment: IND Prod
       steps:
         - step: *deploy-to-ec2

pipelines:
  default:
    - stage: *sonar-cloud-stage
  custom:
    dev:
      - variables:
          - name: PACKAGE_VERSION
            type: string
      - stage: *deploy-to-dev-stage
    staging:
      - variables:
          - name: PACKAGE_VERSION
            type: string
      - stage: *deploy-to-uat-stage
    us-prod:
      - variables:
          - name: PACKAGE_VERSION
            type: string
      - stage: *deploy-to-us-prod-stage
    uk-prod:
      - variables:
          - name: PACKAGE_VERSION
            type: string
      - stage: *deploy-to-uk-prod-stage
    ind-prod:
      - variables:
          - name: PACKAGE_VERSION
            type: string
      - stage: *deploy-to-ind-prod-stage
```

## Summary

- Structure your repository correctly with the necessary files.
- Configure SonarCloud and generate a Sonar token.
- Set up AWS CodeArtifact and configure the necessary environment variables.
- Ensure EC2 instances have SSM agents and proper IAM roles.
- Add environment variables in the CI/CD tool's settings.
- Save and commit the pipeline configuration file (`bitbucket-pipelines.yml`) to your repository.

By following these steps, you can accommodate the provided pipeline file in your repository for an effective CI/CD process.
