---
author: Gaurav Jangid
title: DevOps SFDX CI/CD
date: 2024-07-24
---

# Salesforce DevOps pipelines setup (GitLab)

This document will guide you to setting up pipelines for sfdx projects. These pipelines executes different stages like **PMD** checks, **Dry run** and **Deployment** in different orgs/environment.

Create a Project in GitLab using SFDX template and push the manifest project setup in this repository.

The folder structure of the project should look similar to this

![sfdx folder structure](image-2.png)

### For setting up pipelines we need a yml file named **.gitlab-ci.yml** which is to be created in root of the repository.
---
following is the basic template of <span style="color:blue">.gitlab-ci.yml</span> file.

``` yml
stages:
  - MR_develop_to_QA
  - pmd
  - run_validation
  - QA_deployment
  - QA_data_deployment
  # will add this stage later on

variables:
  PMD_VERSION: "7.0.0"
  GITLAB_API_URL: "https://yourgitlabapiurl/api/v4"
  ASSIGNEE_ID: "329"
  REVIEWER_ID: "329"

#This stage is for creating a scheduled PR from develop to QA branch and is a scheduled stage
create_merge_request:
  stage: MR_develop_to_QA
  image: curlimages/curl:latest
  script:
    - echo "$CI_PROJECT_ID"
    - echo "$GITLAB_API_URL/projects/$CI_PROJECT_ID/merge_requests"
    - echo "${#GITLAB_ACCESS_TOKEN} characters long"
    - >
      curl --request POST --header "PRIVATE-TOKEN: $GITLAB_ACCESS_TOKEN"
      --data "source_branch=develop&target_branch=QA&title=Draft:[DO NOT MERGE]UAT Deployment&remove_source_branch=false&assignee_id=$ASSIGNEE_ID&reviewer_ids=$REVIEWER_ID"
      "$GITLAB_API_URL/projects/$(echo $CI_PROJECT_ID)/merge_requests"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'

#This stage runs a bashScript which further checks for SuppressWarnings and PMD issues in the codebase
pmd_analysis:
  stage: pmd
  image: ubuntu:latest
  before_script:
    - apt-get update
    - apt-get install -y wget default-jdk unzip
    - wget https://github.com/pmd/pmd/releases/download/pmd_releases/$PMD_VERSION-rc4/pmd-dist-$PMD_VERSION-rc4-bin.zip
    - unzip pmd-dist-$PMD_VERSION-rc4-bin.zip -d ~
    - chmod a+x ~/pmd-bin-7.0.0-rc4/shell/pmd-completion.sh
  script:
    - bash bashScripts/countError.sh
    - echo "PMD Script - PR raised"

  rules:
  - if: '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "develop"'

# We are performing dry run on the QA org (using package.xml) before deployment
validateBeforeQACommit:
  image: ubuntu:latest
  stage: run_validation
  before_script:
    - apt-get update
    - apt-get install -y wget default-jdk unzip xz-utils  # Install JDK and wget
    - wget https://developer.salesforce.com/media/salesforce-cli/sf/channels/stable/sf-linux-x64.tar.xz
    - mkdir ~/cli
    - mkdir ~/cli/sf
    - tar xJf sf-linux-x64.tar.xz -C ~/cli/sf --strip-components 1
    - export PATH="$HOME/cli/sf/bin:$PATH"
    - PATH=$HOME/cli/sf/bin:$PATH
    - ~/cli/sf/bin/sf version
    - |
      if [ "$CI_MERGE_REQUEST_TARGET_BRANCH_NAME" == "QA"  ]; then
        echo "  "
        echo "Authenticating in QA sandbox.... ⏱️"
        echo " "
        sf login org jwt --alias deployOrg --client-id ${QA_CONSUMER_KEY} --jwt-key-file accesskey/server.key --set-default --set-default-dev-hub --instance-url https://test.salesforce.com --username ${QA_USERNAME}
        echo " "
        echo "Successfully authenticated in QA sandbox ✅"
        echo " "
      fi
  script:
    - echo "Validation started ... 🔜"
    - echo "sf project deploy start -x manifest/package.xml --dry-run --target-org deployOrg --ignore-warnings --verbose"
    - sf project deploy start -x manifest/package.xml --dry-run --test-level RunLocalTests --target-org deployOrg --verbose
    - echo "Validation Finished 🎯"
  rules:
    - if: $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "QA"

#Deploying all the stuff mentioned in package.xml to QA sandbox
Deploy to QA Sandbox:
  image: ubuntu:latest
  stage: QA_deployment
  before_script:
    - apt-get update
    - apt-get install -y wget default-jdk unzip xz-utils  # Install JDK and wget
    - wget https://developer.salesforce.com/media/salesforce-cli/sf/channels/stable/sf-linux-x64.tar.xz
    - mkdir ~/cli
    - mkdir ~/cli/sf
    - tar xJf sf-linux-x64.tar.xz -C ~/cli/sf --strip-components 1
    - export PATH="$HOME/cli/sf/bin:$PATH"
    - PATH=$HOME/cli/sf/bin:$PATH
    - ~/cli/sf/bin/sf version
    - |
      if [ "$CI_COMMIT_BRANCH" == "QA"  ]; then
        echo "  "
        echo "Authenticating in QA sandbox.... ⏱️"
        echo " "
        sf login org jwt --alias deployOrg --client-id ${QA_CONSUMER_KEY} --jwt-key-file accesskey/server.key --set-default --set-default-dev-hub --instance-url https://test.salesforce.com --username ${QA_USERNAME}
        echo " "
        echo "Successfully authenticated in QA sandbox ✅"
        echo " "
      fi
  script:
    - echo "Deployment started ... 🔜"
    - echo "sf project deploy start -x manifest/package.xml --target-org deployOrg --ignore-warnings --verbose"
    - sf project deploy start -x manifest/package.xml --target-org deployOrg --test-level RunLocalTests --verbose
    # - sf project deploy start --source-dir force-app/ --test-level RunAllTestsInOrg --target-org deployOrg --ignore-warnings --verbose
    - echo "Deployment Finished 🎯"
  rules:
    - if: $CI_COMMIT_BRANCH == "QA"

# Similar to QA we can perform deployment in UAT and production as well
```

In the above xml some variables are being used which are predefined but some of them are defined in the ci-cd settings of the project repository which can be changed by a maintainer of the repository

The above xml refers to a bashScript and here is a example for that

``` bash
#!/bin/bash
variable_to_search="@SuppressWarnings"

find . -type f ! -name "countError.sh" -exec grep -iq "$variable_to_search" {} \; -exec grep -iHn "$variable_to_search" {} \;
echo "--------------------------------------------------------------------------------------"
swcount=$(find . -type f ! -name "countError.sh" -exec grep -iq "$variable_to_search" {} \; -exec grep -iHn "$variable_to_search" {} \; | wc -l)
echo "Total suppresss warning count: $swcount"

# we can update this suppressWarning count according to the requirement but it should not exceed the count of 5 (only suppress the pmd errors when required)
if [ $swcount -gt 0 ]; then
   echo "SuppressWarnings found 🚨"
   echo " "
   echo "Job Failed, Kindly remove the SupressWarning!"
   exit 125
else
   echo "No SupressWarnings found ✅"
fi

# Check for pmd errors in files present in force-app folder using rules defined in ruleset.xml file
PMD_ERRORS=$(~/pmd-bin-7.0.0-rc4/bin/pmd check -d force-app -R pmd/ruleset.xml -f text | grep 'force-app/')
PMD_ERRORS_COUNT=0
PMD_ERRORS_COUNT=$(echo $PMD_ERRORS | grep -o 'force-app' | wc -l)
echo "Error count is: $PMD_ERRORS_COUNT"
PMD_ERRORS=$(echo "$PMD_ERRORS" | sed 's/force-app/\n\/force-app/g')

if [ $PMD_ERRORS_COUNT -gt 0 ]; then
   echo "Errors are $PMD_ERRORS"
   echo " "
   echo "Total PMD Analysis Error count: $PMD_ERRORS_COUNT"
   echo " "
   echo "Job Failed, Kindly resolve pmd analysis errors! 📍"
   exit 125
else
   echo "PMD Analysis Passed. No Error Found 🎯"
fi

```

The above bash script refers to ruleset.xml which contains all the PMD rules. We can also remove or add some additional rules and also modify rules by changing their limits and properties.

Refer to this link for more : [PMD Rules](https://pmd.github.io/pmd/pmd_rules_apex.html)
---
Here is a standard ruleset file.

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<ruleset xmlns="http://pmd.sourceforge.net/ruleset/2.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Default ruleset used by the CodeClimate Engine for Salesforce.com Apex" xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 https://pmd.sourceforge.io/ruleset_2_0_0.xsd">
   <description>
      Default ruleset used by the Code Climate Engine for Salesforce.com Apex
   </description>

   <!-- Documentation -->
   <rule ref="category/apex/documentation.xml/ApexDoc" />

   <!-- Best Practices -->
   <rule ref="category/apex/bestpractices.xml/ApexAssertionsShouldIncludeMessage" />
   <rule ref="category/apex/bestpractices.xml/ApexUnitTestClassShouldHaveAsserts" />
   <rule ref="category/apex/bestpractices.xml/ApexUnitTestClassShouldHaveRunAs" />
   <rule ref="category/apex/bestpractices.xml/ApexUnitTestMethodShouldHaveIsTestAnnotation" />
   <rule ref="category/apex/bestpractices.xml/ApexUnitTestShouldNotUseSeeAllDataTrue" />
   <rule ref="category/apex/bestpractices.xml/AvoidGlobalModifier" />
   <rule ref="category/apex/bestpractices.xml/AvoidLogicInTrigger" />
   <rule ref="category/apex/bestpractices.xml/DebugsShouldUseLoggingLevel" />
   <rule ref="category/apex/bestpractices.xml/UnusedLocalVariable" />

   <!-- Code Style -->
   <rule ref="category/apex/codestyle.xml/ClassNamingConventions" />
   <rule ref="category/apex/codestyle.xml/FieldDeclarationsShouldBeAtStart" />
   <rule ref="category/apex/codestyle.xml/FieldNamingConventions" />
   <rule ref="category/apex/codestyle.xml/ForLoopsMustUseBraces" />
   <rule ref="category/apex/codestyle.xml/FormalParameterNamingConventions" />
   <rule ref="category/apex/codestyle.xml/IfElseStmtsMustUseBraces" />
   <rule ref="category/apex/codestyle.xml/IfStmtsMustUseBraces" />
   <rule ref="category/apex/codestyle.xml/LocalVariableNamingConventions" />
   <rule ref="category/apex/codestyle.xml/MethodNamingConventions" />
   <rule ref="category/apex/codestyle.xml/OneDeclarationPerLine" />
   <rule ref="category/apex/codestyle.xml/PropertyNamingConventions" />
   <rule ref="category/apex/codestyle.xml/WhileLoopsMustUseBraces" />

   <!-- Design -->
   <rule ref="category/apex/design.xml/AvoidDeeplyNestedIfStmts" />
   <rule ref="category/apex/design.xml/CognitiveComplexity" >
      <properties>
        <property name="classReportLevel" value="60" />
        <property name="methodReportLevel" value="30" />
      </properties>
   </rule>
   <rule ref="category/apex/design.xml/CyclomaticComplexity"/>
   <rule ref="category/apex/design.xml/ExcessiveClassLength" />
   <rule ref="category/apex/design.xml/ExcessiveParameterList" />
   <rule ref="category/apex/design.xml/ExcessivePublicCount" />
   <rule ref="category/apex/design.xml/NcssConstructorCount" />
   <rule ref="category/apex/design.xml/NcssMethodCount" />
   <rule ref="category/apex/design.xml/NcssTypeCount" />
   <rule ref="category/apex/design.xml/StdCyclomaticComplexity" />
   <rule ref="category/apex/design.xml/TooManyFields" />
   <rule ref="category/apex/design.xml/UnusedMethod" />

   <!-- Error Prone -->
   <rule ref="category/apex/errorprone.xml/ApexCSRF" />
   <rule ref="category/apex/errorprone.xml/AvoidDirectAccessTriggerMap" />
   <rule ref="category/apex/errorprone.xml/AvoidHardcodingId" />
   <rule ref="category/apex/errorprone.xml/AvoidNonExistentAnnotations" />
   <rule ref="category/apex/errorprone.xml/EmptyCatchBlock" />
   <rule ref="category/apex/errorprone.xml/EmptyIfStmt" />
   <rule ref="category/apex/errorprone.xml/EmptyStatementBlock" />
   <rule ref="category/apex/errorprone.xml/EmptyTryOrFinallyBlock" />
   <rule ref="category/apex/errorprone.xml/EmptyWhileStmt" />
   <rule ref="category/apex/errorprone.xml/InaccessibleAuraEnabledGetter" />
   <rule ref="category/apex/errorprone.xml/MethodWithSameNameAsEnclosingClass" />
   <rule ref="category/apex/errorprone.xml/OverrideBothEqualsAndHashcode" />

   <!-- Performance -->
   <rule ref="category/apex/performance.xml/AvoidDebugStatements" />
   <rule ref="category/apex/performance.xml/EagerlyLoadedDescribeSObjectResult" />
   <rule ref="category/apex/performance.xml/OperationWithLimitsInLoop" />

   <!-- Security -->
   <rule ref="category/apex/security.xml/ApexCRUDViolation" />
   <rule ref="category/apex/security.xml/ApexBadCrypto" />
   <rule ref="category/apex/security.xml/ApexInsecureEndpoint" />
   <rule ref="category/apex/security.xml/ApexOpenRedirect" />
   <rule ref="category/apex/security.xml/ApexSharingViolations" />
   <rule ref="category/apex/security.xml/ApexSOQLInjection" />
   <rule ref="category/apex/security.xml/ApexSuggestUsingNamedCred" />
   <rule ref="category/apex/security.xml/ApexXSSFromEscapeFalse" />
   <rule ref="category/apex/security.xml/ApexXSSFromURLParam" />

</ruleset>

```

The above **.gitlab-ci.yml** file contains deployment stages as well which requires a server key. This key is related to a certificate which is generated in our local device using some commands.

For deploying code to a org we require a connected app to be created. In this connected app we upload the certificate file.

Refer to this for creating certificate and key : [link](https://developmentisinprogress.com)

while deploying the code we perform authentication using these key and certificate files.

The file is located in accesskey/server.key

``` key
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEAyTylXbr1WcZJ9Ue5GZ17VUJfkIstmEsetItcZanyRxo/MeBE
E/pAgJ1UE9InpVzRjgGZgsp1eeQP5FpkpmfViXq17gJ3o7TptbgYb4CUF+bIwOeB
daEJm0fH9KE1UE+zMfK+hH+W8LULB85cUh+XTXBy072wR/EpqFzzJVi2bcMKaAfn
jJ3M64sgnY0gg1X3Lf323Py/29K4DwvhRAXkIeYvR1scYtiHEi9dXIEFHrCaO9Xl
CMuYORpA/C0K90W9WAaWUu2LEZjCNo65Uxg/G3XwLp/IoboNt9TgozLcXTUENmZP
WpNJ/kGmVdI7u.............................ABAoIBAQCbS3jWB2Q1vobu
iUX31YcK+aij9DPgdgAhFfbc4bR9WGzyQX5n6FSErGTo0uITg7iXnPZpb3IO+Yem
EnffzvuEF7V0EmVfaUp+GK6KxbADbe74pu9J5ldyhIdzakCpVIgWAJ0zxP6njR8R
DBhEq+poMWH7s0PpkNsz+V8OMBLASkaarnWMuaNIf8wQbkD543MeXTHOJHN9XnAG
TmQ2zRCfe00X9SWhkeuF8J9oATS0G252kNfciLX1BeQheGgtAOWXS2R7yOAUCxEr
qMKR143gn1upxElesuVOl+eHlZuCd+L9Zcgxx2RgtLi2Sb2YWDTMdnSU+YMICVDD
ykuD2lYRAoGBAPrLti7u.................LVLkKh7LRDrqvJFKiL3dZjNnINe
nGAPC9qabIq3abNw5yiJ0Djm403Pkg1kRcmbRqyFrgGXZSFGhdAH3tNZ3fuCgXr4
m03FNVa1sdsQeyJRA1RXi4oNCtapSFXuU5rrilumwVIPp9Ab9nTmIA+VAoGBAM1p
qmMpheEPyKA6FrsemkM/UYFuUo5ztICO/DDzHmQPv9ZaojAfXQXvJiLWz1sW8Mww
Hb40ese4zsmhW6JHF8UWPf+fSlbuRIyUm+9UWuqMT96OhwQSiRz3PQ0JJpX6sVdB
20f0CMbKVDU+aX4f/w171xm2kCimqKuT5zLUoguNAoGAARczsp6NSraHTrXyzhAH
b9cDyaYLegAXIRS1SJpuZXqIA2MZhCS8L1rBZXD4ed30Urhx+OYohqbSZDs72l0O
RKMtOoDWUeFzqiJhYJc4K9kfMwHNSFiebEcoRTlDDb3Tm0GY7xv2WmJWIrrXuq/v
oyPVFTLzckTpKOmGN9nsV8UCgYEAxaYqwQevPoPFdAbw4FPt8071yiixX0iOIjVz
DFaio31UBZln/............................2uCbk2zSPIkYyV7SOIz4gSK
ripTUfECgYEA2WGNHmhbB1ovhGZWzcOcGxb7GL95XLDWKdHW67GJlBlf3RzQgHTX
IEA+tgKeZZ7BxcDDf12MoGbZYEpohxCyWZYRzbQI/xzXIpMRFmQ5TIDsxd63UY8C
wwsoXM1G4p0lVco0qprR5b/cF39JOor1zmfXmHGU1181FxDAKQd/GIg=
-----END RSA PRIVATE KEY-----
```
