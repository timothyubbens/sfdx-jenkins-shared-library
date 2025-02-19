# Salesforce DX - Jenkins Shared Library [![Build Status](https://travis-ci.com/claimvantage/sfdx-jenkins-shared-library.svg?token=3fsbWz1ejyryiEvBSie2&branch=master)](https://travis-ci.com/claimvantage/sfdx-jenkins-shared-library)

## Contents
* [Why?](#why)
* [Prerequisites](#prerequisites)
* [Pipelines](#pipelines)
  * [sfdxBuildPipeline](#sfdxBuildPipeline)
* [Steps](#steps)
  * [createScratchOrg](#createScratchOrg)
  * [deleteScratchOrg](#deleteScratchOrg)
  * [installPackage](#installPackage)
  * [processHelp](#processHelp)
  * [pushToOrg](#pushToOrg)
  * [retrieveExternals](#retrieveExternals)
  * [runApexTests](#runApexTests)
  * [runLint](#runLint)
  * [runLightningTests](#runLightningTests)
  * [shWithResult](#shWithResult)
  * [shWithStatus](#shWithStatus)
  * [withOrgsInParallel](#withOrgsInParallel)
* [Multiple Orgs](#multiple)
* [Org Bean](#org)
* [Running Library Tests Locally](#runningLibraryTestsLocally)

<a name="why"></a>
## Why?

The two aims of this library are, for [Salesforce DX](https://www.salesforce.com/products/platform/products/salesforce-dx/) Jenkins builds:

* To avoid the duplication of 150+ lines of `Jenkinsfile` logic across builds (at ClaimVantage we have dozens of builds) so a reliable pattern can be applied and maintained.

  This is accomplished by providing custom pipeline steps that hide some of the detail.
  A default pipeline that sits on top of these steps is also provided
  and is **recommended** for projects that fit its pattern.
  
* To make the process of testing against various org configurations - e.g. person Accounts turned on or Platform Encryption turned on - simple, and not require additional builds to be setup.

  When multiple
  [Scratch Org Definition File](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file.htm)
  files are provided that match a regular expression, parallel builds are done using scratch orgs created from the files.
  
The resulting Jenkinsfile can be as simple as:
```groovy
#!groovy
@Library('sfdx-jenkins-shared-library')_

sfdxBuildPipeline()
```
Note the required `_` for this case.
  
For some background information including how to hook up this library, see e.g.
[Extending your Pipeline with Shared Libraries, Global Functions and External Code](https://jenkins.io/blog/2017/06/27/speaker-blog-SAS-jenkins-world/). Libraries are pulled directly from Git for
each new build, so setup is simple.

<a name="prerequisites"></a>
## Prerequisites

### Unix Only

Makes use of shell scripting so only Unix platforms are supported.

### Jenkins

Use a recent "Long Term Support" (LTS) version of [Jenkins](https://jenkins.io/). Add these:

* [Credentials Binding Plugin](https://jenkins.io/doc/pipeline/steps/credentials-binding/)
* [Workspace Cleanup plugin](https://jenkins.io/doc/pipeline/steps/ws-cleanup/)
* [SSH Agent](https://plugins.jenkins.io/ssh-agent)
    * Only required when the [retrieveExternals](#retrieveExternals) or [processHelp](#processHelp) step is used

### Tools

Requires [Salesforce SFDX CLI](https://developer.salesforce.com/docs/atlas.en-us.sfdx_setup.meta/sfdx_setup/sfdx_setup_install_cli.htm) to be installed where Jenkins is running (and also a Git client). If Git externals are being used, also requires [develersrl/git-externals](https://github.com/develersrl/git-externals).

This library has only been tested with GitHub.

### Jenkins Environment Variables

These must be set up for all the stages to work.

| Name | Description | Example |
|:-----|:------------|:--------|
| CONFLUENCE_CREDENTIAL_ID<sup>[1]</sup> | Confluence username/password credentials stored in Jenkins in "Credentials" under this name. Only used by the ClaimVantage proprietary **processHelp** step. | to-confluence |
| GITHUB_CREDENTIAL_ID<sup>[2]</sup> | A GitHub generated private key **and** username stored in Jenkins in "Credentials" under this name.| to-github |

[1] Used to extract help pages from Confluence.

[2] Used to retrieve Git externals.

### Dev Hub Authentication

This library connects to the default Dev Hub configured on the build agent by using a JSON Web Token ([JWT](https://jwt.io/)). See [Authorize an Org Using the JWT-Based Flow](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm#sfdx_dev_auth_jwt_flow) for how to set that up.

Use the Client Id (the Consumer Key on the Connected App), the [JWT Key file](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_key_and_cert.htm) (private key that signed the certificate configured on Connected App), from the JWT setup process in the following steps. Also a user must be setup in the Dev Hub for Jenkins e.g. jenkins@acdxgs0hub.org, and that username is also needed in the following steps.

The steps are:
1. Log into the  build agent as the user that Jenkins run as (usually a user named Jenkins).
2. Save the JWT Key File in a folder that won't get deleted by Jenkins builds e.g. `/Users/jenkins/JWT/server.key`. The server.key file content should look something like this:
```
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```
3. Manually authenticate access to the Dev Hub using the command below:
```bash
sfdx force:auth:jwt:grant \
--clientid 04580y4051234051 \
--jwtkeyfile /Users/jenkins/JWT/server.key \
--username jenkins@acdxgs0hub.org \
--setdefaultdevhubusername \
--setalias my-hub-org
```

This authentication will stay in place until the certificate created as part of the setup expires.

<a name="pipelines"></a>
## Pipelines

<a name="sfdxBuildPipeline"></a>
### sfdxBuildPipeline

This is a [ready-made pipeline](vars/sfdxBuildPipeline.groovy) - **recommended** that you start with this - that runs these stages using both the steps listed in the [Steps](#steps) section below and standard steps:

```groovy
stage("help") {...}                 // Only runs if help or helps beans are defined         
stage("checkout") {...}
stage("after checkout") {...}       // Only runs if afterCheckoutStage closure is defined
withOrgsInParallel() {
    stage("org create") {...}
    try {
        stage("org install") {...}      // Only runs if package or packages beans are defined
        stage("org before push") {...}  // Only runs if beforePushStage closure is defined
        stage("org push") {...}
        stage("org before test") {...}  // Only runs if beforeTestStage closure is defined
        stage("org test") {...}
        stage("org after test") {...}   // Only runs if afterTestStage closure is defined
    } finally {
        stage("org delete") {...}
    }
}
stage("publish")  {...}
stage("clean") {...}
stage("finalStage") {...}           // Only runs if finalStage closure is defined
```
To use it, your `Jenkinsfile` should look like this (and you will need
[Scratch Org Definition File](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file.htm)
files that match the regular expression):
```groovy
#!groovy
@Library('sfdx-jenkins-shared-library')
import com.claimvantage.sjsl.Help
import com.claimvantage.sjsl.Package

sfdxBuildPipeline(
    glob: 'config/project-scratch-def.*.json',
    help: new Help('cx', '33226968', 'extras-help'),
    packages: [
        new Package('cve', env.'cve.package.password.v12'),
        new Package('cvab', env.'cvab.package.password.v12')
    ]
)
```

Edit the Help and Package details to reflect the specific project.

You can use strings instead of IDs in sfdx-project.json to install packages, but if you’re comfortable using packaging IDs and prefer to not use aliases you can set IDs as input.
The first parameter of the Package constructor may be an alias for the Package or the Package Id.
See [Package Alias](#packageAliases) for details.

To build a package that has multiple configurations that require additional components or data to be setup (before the tests are run):
```groovy
#!groovy
@Library('sfdx-jenkins-shared-library')
import com.claimvantage.sjsl.Help
import com.claimvantage.sjsl.Package

sfdxBuildPipeline(

    glob: 'config/project-scratch-def*.json',

    help: new Help('ch', '22315079', 'claims-help'),

    beforeTestStage: { org ->
        echo "${org.name} before test stage"
        switch (org.name) {
            case 'content-notes':
                echo "${org.name} deploying extra components"
                shWithResult "sfdx force:source:deploy --sourcepath ${env.WORKSPACE}/config-components/content-notes --json --targetusername ${org.username}"
                break
            case 'accommodations':
                echo "${org.name} installing Accommodations"
                installPackage(org: org, package: new Package('cvawa', env.'cvawa.package.password.v12'))
                break
            case 'platform-encryption':
                echo "${org.name} setting up encryption"
                sh "sfdx force:user:permset:assign --permsetname Encryption --targetusername ${org.username}"
                sh "sfdx force:data:record:create --sobjecttype TenantSecret --values 'Description=Test' --targetusername ${org.username}"
                shWithResult "echo 'cve.SetupController.updateDefaultFieldEncryptedFlags(true);' | sfdx force:apex:execute --json --targetusername ${org.username}"
                break
        }
    },

    afterTestStage: { org ->
        sh "npm install"
        sh "./node_modules/karma/bin/karma start karma.conf.js"
    }
)
```

The named values available are:

* _afterCheckoutStage_

  An (optional) closure that is executed immediately after the checkout stage.
  See the [Org Bean](#org) section below.
  
  This example executes lint validation against a specific folder, which could prevent scratch org creation if it fails.
  ```groovy
  afterCheckoutStage: {
      sh "sfdx force:lightning:lint ./path/to/lightning/components/"
  }
  ```
* _afterOrgCreateStage_

  An (optional) closure that is executed immediately after the org create stage. The `org` is passed in to this.
  See the [Org Bean](#org) section below.

  This is good point to perform operations before the possible packages installations.

  For example, if you need to perform any operation on the new scratch org _before_ any package is installed:
  ```groovy
  afterOrgCreateStage: { org ->
      sh "./setupOrg.sh ${org.username}"
  }
  ```

* _beforePushStage_

  An (optional) closure that is executed immediately before the push stage. The `org` is passed in to this.
  See the [Org Bean](#org) section below.

  This is good point to insert extra content into the source tree, but that content has to apply to all org configurations
  as a common copy of the source tree is used.

* _beforeTestStage_

  An (optional) closure that is executed immediately before the test stage. The `org` is passed in to this.
  See the [Org Bean](#org) section below.
  
  This example executes some Apex code included in the pushed code (in this case setting up a role):
  ```groovy
  beforeTestStage: { org ->
      sh "echo 'cveep.UserBuilder.ensureRole();' | sfdx force:apex:execute --targetusername ${org.username}"
  }
  ```
 
* _cron_

  Optional. A map of branch name to [cron expression](https://jenkins.io/doc/book/pipeline/syntax/#cron-syntax).
  This allows a branch (or branches) to be built regularly, in addition to when changes are made in Git.
   
  This example builds the master branch every day between midnight and 5:59 AM:
  ```groovy
  cron: ['master': 'H H(0-5) * * *']
  ```

* _daysToKeepPerBranch_

  Optional (defaults to 7 days). Defines how long builds are kept in Jenkins.
  This example keeps the master branch for 7 days.
  ```groovy
  daysToKeepPerBranch: ['master': 7]
  ```

* _glob_

  The matching pattern
  (see [Ant style pattern](https://ant.apache.org/manual/dirtasks.html#patterns) for details - * matches zero or more characters, ? matches one character)
  used to find the
  [Scratch Org Definition File](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file.htm)
  files. Each matched file results in a separate parallel build.
  The default value is "config/project-scratch-def.*.json"; this assumes that an extra part will be inserted
  into the file names and that part is used as a name for the parallel work.
  
  There are two ways to specify the matching pattern:

  - to use the same matching pattern for all branches, just supply the matching pattern as a string
  - to vary the matching pattern by branch, specify a map of branch names to matching patterns

  A particularly useful map to use is this one, that identifies all org configurations for the master branch, but only one org configuration for other branches:
  ```groovy
  // Evaluate so a serializable string is used in the pipeline (the withDefault object is not serializable)
  // See https://issues.jenkins-ci.org/browse/JENKINS-38186
  glob: ['master': 'config/project-scratch-def*.json'].withDefault{'config/project-scratch-def.json'}[env.BRANCH_NAME],
  ```
  and so reduces the number of Scratch Orgs consumed when multiple branches are being worked on at the same time.

* _help_ (or _helps_)

  **Disclaimer**: Do not include this as it contains dependencies on private ClaimVantage repositories that we don't plan to release to the community.
  
  Reference a simple bean object (or an array of those objects) that holds the values needed to extract, process, and commit the help.
  When left out, no help processing is done.
  
* _keepOrg_

  Optional. When set to true, the scratch orgs are not deleted and instead login credentials are output via an **echo** in the stage.
  This allows the orgs to be examined after the build to e.g. reproduce test failures manually.
  The scratch orgs can then be manually deleted via the "Active Scratch Orgs" tab of the Dev Hub,
  or just left to expire.

* _keepWs_

  Optional. When set to true, the workspace is not deleted. This allows data such as raw test test result XML
  files to be examined after the build.

* _package_ (or _packages_)

  Reference a simple bean object (or an array of those objects) that holds the values needed to install existing managed package versions.
  When left out, no package installation is done.

* _stagger_

  Optional (defaults to 15 seconds).
  This is the number of seconds to delay before the next parallel set of steps is started. 
  The aim is to smooth out the load a little both on the Jenkins machine and at the Salesforce side
  by staggering the the execution of the parallel logic.

<a name="steps"></a>
## Steps

The general pattern to use these steps is this, where the `withOrgsInParallel` step passes the `org` value into the closure:
```groovy
#!/usr/bin/env groovy
@Library('sfdx-jenkins-shared-library')
import com.claimvantage.sjsl.Help
import com.claimvantage.sjsl.Org
import com.claimvantage.sjsl.Package

node {
    stage("checkout") {
        ...
    }
    withOrgsInParallel() { org ->
        stage("${org.name} create") {
            createScratchOrg org
        }
        ...
    }
    stage("publish") {
        ...
    }
}
```

<a name="createScratchOrg"></a>
### createScratchOrg

[Creates a scratch org](vars/createScratchOrg.groovy)
and adds values relating to that to the supplied `org` object for use by later steps. This step has to come before most other steps. The org is created with the minimum duration value of `--durationdays 1` as the number of active scratch orgs
is limited, and failing builds might not get to their **deleteScratchOrg** step. You can change the duration day by changing the Org object. Details of the created org are output into the log via an **echo** step. It sets job name as the alias for the created scratch org.

* _org_

  Required. An instance of Org that has it's `projectScratchDefPath` property set.

<a name="deleteScratchOrg"></a>
### deleteScratchOrg

[Deletes a scratch org](vars/deleteScratchOrg.groovy) 
identified by values added to the [Org](src/com/claimvantage/jsl/Org.groovy) object by **createScratchOrg**. This step has to come after most other steps.

* _org_

  Required. An instance of Org that has been populated by **createScratchOrg**.

<a name="installPackage"></a>
### installPackage

[Installs a package](vars/installPackage.groovy)
into a scratch org. Package installs typically take 2 to 20 minutes depending on the package size.
The Package name and version are output via an **echo** in the stage.

* _org_

  Required. An instance of Org that has been populated by **createScratchOrg**.
  
* _package_

  Required. An instance of the [Package](src/com/claimvantage/sjsl/Package.groovy) bean object
  whose properties identify the package version to install.

<a name="packageAliases"></a>
#### Package Aliases

See [Salesforce Project Configuration File for Packages](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev2gp_config_file.htm) for details.
Use the example below to configure package aliases; the benefit is that the package version Id (that typically changes over time) is kept out of the Jenkinsfile that should not need to change over time.

The general pattern to use alias is to configure it on your sfdx-project.json file:
```JSON
{
    "packageAliases": {
        "cve": "04t50000000AkkX",
        "cvab": "04t0V000000xEEs"
    }
}
```

<a name="processHelp"></a>
### processHelp

**Disclaimer**: Do not use this step as it contains dependencies on private ClaimVantage repositories that we don't plan to release to the community.

This is a [ClaimVantage proprietary stage](vars/processHelp.groovy) that extracts
help content from Confluence, processes that content and then adds the content to Git so that
it can be pulled into a package via Git externals.

* _branch_

  The branch name for which this step runs.
  The default value is "master".

* _help_

  Required. An instance of the [Help](src/com/claimvantage/sjsl/Help.groovy) bean object
  whose properties identify the help information.
  
<a name="pushToOrg"></a>
### pushToOrg

[Pushes](vars/pushToOrg.groovy) the components into a scratch org.

* _org_

  Required. An instance of Org that has been populated by **createScratchOrg**.

<a name="retrieveExternals"></a>
### retrieveExternals

If a `git_externals.json` file is in the repository root,
[uses git-externals](vars/retrieveExternals.groovy) to pull in that content.
If no file is present, the step does nothing (and git-externals does not have to be installed).

<a name="runApexTests"></a>
### runApexTests

[Runs Apex tests](vars/runApexTests.groovy) for an org and puts the test results in a unique folder
based on the name of the `org` object.
The test class names are also prefixed by that name so that when multiple orgs are tested,
the test results are presented separated by the name.

To workaround frequent EAI_AGAIN errors while the tests are running, the current implementation polls
to check whether the tests have finished rather than making the blocking SFDX call.

* _org_

  Required. An instance of Org that has been populated by **createScratchOrg**.

<a name="runLightningTests"></a>
### runLightningTests

[Runs Lightning tests](vars/runLightningTests.groovy) for an org and puts the test results in a unique folder
based on the name of the `org` object.
The test class names are also prefixed by that name so that when multiple orgs are tested,
the test results are presented separated by the name.

The [Lightning Testing Service](https://github.com/forcedotcom/LightningTestingService) **must be present in the org** e.g. by
making it part of the SFDX project.

* _org_

  Required. An instance of Org that has been populated by **createScratchOrg**.

* _appName_

  The name of the Lightning app used to test the application. For example "Test.app".

* _configFile_

  Optional. The path to a test configuration file to configure WebDriver and other settings.
  There isn't much official documentation on this; this
  [Salesforce Stackexchange answer](https://salesforce.stackexchange.com/questions/200451/how-to-run-lightning-test-service-lts-from-jenkins-hosted-on-aws-against-e-g)  provides some information. An example file is:
  ```JSON
  {  
      "webdriverio":{  
          "desiredCapabilities": [{  
              "browserName": "chrome"
          }],
          "host": "hub.browserstack.com",
          "port": 80,
          "user": "username",
          "key": "password"
      }
  }
  ```
<a name="runLint"></a>
### runLint

[Runs force:lightning:lint Tool in Parallel](vars/runLint.groovy) on multiple folders that contains Lightning components.

* _folders_

  Required. A list of folders that needs to be validated
  
  It might look like this:
  ```groovy
  def LINT_FOLDERS = ["./sfdx-source/wiz/main/aura", "./sfdx-source/int/main/aura"]
  runLint(folders:LINT_FOLDERS)
  ```

<a name="shWithResult"></a>
### shWithResult

Typically not directly used as this is a building block for other steps.

[Runs a shell command](vars/shWithResult.groovy) assumed to be an SFDX command that produces JSON output (typically via the `--json` argument), checks if the result is 0 (if it is not then it fails), and parses the output using [readJSON](https://jenkins.io/doc/pipeline/steps/pipeline-utility-steps/#readjson-read-json-from-files-in-the-workspace) step, returning the result object.

<a name="shWithStatus"></a>
### shWithStatus

Typically not directly used as this is a building block for other steps.

[Runs a shell command](vars/shWithStatus.groovy) and checks if the result is 0 (if it is not then it fails).

<a name="withOrgsInParallel"></a>
### withOrgsInParallel

Finds matching
[Scratch Org Definition File](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file.htm)
files, and for each one uses the Jenkins Pipeline **parallel** step to [execute
the nested steps](vars/withOrgsInParallel.groovy). This allows multiple org configurations to be handled at the same time.

* _glob_

  The matching pattern
  (see [Ant style pattern](https://ant.apache.org/manual/dirtasks.html#patterns) for details - * matches zero or more characters, ? matches one character)
  used to find the
  [Scratch Org Definition File](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file.htm)
  files. Each matched file results in a separate parallel build.
  The default value is "config/project-scratch-def.*.json"; this assumes that an extra part will be inserted into the file names and that part is used as a name for the parallel work.
  
  There are two ways to specify the matching pattern:

  - to use the same matching pattern for all branches, just supply the matching pattern as a string
  - to vary the matching pattern by branch, specify a map of branch names to matching patterns

  A particularly useful map to use is this one, that identifies all org configurations for the master branch, but only one org configuration for other branches:
  ```groovy
  // Evaluate so a serializable string is used in the pipeline (the withDefault object is not serializable)
  // See https://issues.jenkins-ci.org/browse/JENKINS-38186
  glob: ['master': 'config/project-scratch-def*.json'].withDefault{'config/project-scratch-def.json'}[env.BRANCH_NAME],
  ```
  and so reduces the number of Scratch Orgs consumed when multiple branches are being worked on at the same time.

* _stagger_

  Optional (defaults to 15 seconds).
  This is the number of seconds to delay before the next parallel set of steps is started. 
  The aim is to smooth out the load a little both on the Jenkins machine and at the Salesforce side
  by staggering the the execution of the parallel logic.

<a name="multiple"></a>
## Multiple Orgs

Each scratch org that is created and used (in parallel) is defined by a [Scratch Org Definition File](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file.htm). The number of scratch orgs you can create per day is [limited by Salesforce](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs.htm).

A single file, called `project-scratch-def.json`, might look like this:

```JSON
{
    "orgName": "Jenkins Claims - (default)",
    "edition": "Developer",
    "namespace": "cve",
    "settings": {
        "orgPreferenceSettings": {
            "s1DesktopEnabled": true,
            "disableParallelApexTesting": true
        }
    }
}
```
Adding a second file called `project-scratch-def.person-accounts.json` (note the added `features` line) that looks like this:
```JSON
{
    "orgName": "Jenkins Claims - (person-accounts)",
    "edition": "Developer",
    "namespace": "cve",
    "features": ["PersonAccounts"],
    "settings": {
        "orgPreferenceSettings": {
            "disableParallelApexTesting": true
        }
    }
  }
```
 will result in the org-specific steps running in parallel for both orgs. Adding more files will result in more parallel work.
 
Not everything required can be configured via the `project-scratch-def.json`, so there is also an extension point attribute in the **sfdxBuildPipeline** called _beforeTestStage_ where a closure can be added that executes arbitrary logic and is passed an `org`. Here is an example of using that extension point, in this case to setup platform encryption:
```groovy
sfdxBuildPipeline(
    beforeTestStage: { org ->
        if (org.name == 'platform-encryption') {
            sh "sfdx force:user:permset:assign --permsetname Encryption --targetusername ${org.username}"
            sh "sfdx force:data:record:create --sobjecttype TenantSecret --values 'Description=Test' --targetusername ${org.username}"
            shWithResult "echo 'cve.SetupController.updateDefaultFieldEncryptedFlags(true);' | sfdx force:apex:execute --json --targetusername ${org.username}"
        }
    }
)
```

<a name="org"></a>
## Org Bean

The attributes of the [Org](src/com/claimvantage/sjsl/Org.groovy) object (in order of usefulness) are:

| Attribute | Description |
|:----------|:------------|
| `name` | Name of the scratch org. It is the unique part of the Scratch Org Definition File name extracted from the `projectScratchDefPath`. |
| `username` | Used to direct SFDX commands to the right org after the org has been created. |
| `password` | Allows interactive login if the org is kep after the build. |
| `instanceUrl` | Allows interactive login if the org is kep after the build. |
| `orgId` | Perhaps useful for looking up the scratch org in e.g. the Dev Hub. |
| `projectScratchDefPath` | The path to the specific Scratch Org Definition File. |


<a name="runningLibraryTestsLocally"></a>
### Running Library Tests Locally
Test are automatically executed using Continuous Integration through [Travis CI](https://travis-ci.com/)).

Install [Gradle](https://gradle.org) [4.0.1](https://gradle.org/releases/#4.0.1) (matches the version used in Travis CI (recommend using [SDKMAN!](https://sdkman.io/) listed on [Gradle Installation guide](https://gradle.org/install/)).

If you are using [Visual Studio code](https://code.visualstudio.com/) you can run the [default test task provided](.vscode/tasks.json).

Otherwise you can run the following in project root:
```bash
gradle assemble
gradle check
```
